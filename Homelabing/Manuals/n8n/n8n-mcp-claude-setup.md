# Connecting n8n's MCP server to Claude Code

Companion to `n8n-cloudflare-tunnel-setup.md`. That doc covers getting n8n running on k3s behind a Cloudflare Tunnel with Access in front. This one covers letting Claude Code actually talk to it.

Final working state: Claude Code connects to n8n's instance-level MCP server over HTTP, authenticating past Cloudflare Access with a service token and past n8n with a bearer token. 34 tools exposed.

## The core problem

Cloudflare Access is built for humans in browsers. Someone hits the URL, gets bounced to a login page, enters an email code, comes back with a session cookie.

An MCP client has none of that. No browser, no inbox, no way to complete the flow. So every request Claude Code made got a `302` to the Access login page, and the client reported `Unexpected content type: text/html` because it expected JSON-RPC and got an HTML login form.

Every failure in this setup traced back to that, in one form or another.

## Why the claude.ai connector route does not work

n8n's "Connect a client" dialog offers OAuth as the recommended path, and claude.ai has an n8n connector in its directory. That combination cannot work behind Access.

Claude.ai's OAuth flow uses Dynamic Client Registration, so it tries to register itself with n8n's sign-in service before anything else. That registration request hits the Access login wall and dies, producing:

```
Couldn't register with n8n's sign-in service. [...] reference: "ofid_..."
```

The connector UI has nowhere to attach Cloudflare service token headers, so there is no way to get the registration request through. Adding a service token policy does not help here, because the connector cannot send the headers the policy checks for.

Conclusion: use static token auth via `claude mcp add`, not the directory connector. The connector was removed from claude.ai settings entirely to stop it competing with the working entry.

## 1. Cloudflare service token

A service token is the supported way to let a non-interactive client past Access without weakening the email gate for humans.

Zero Trust → **Access → Service Auth → Service Tokens → Create**. Name it something identifiable like `claude-code`. The Client ID and Client Secret are shown once. The secret is not retrievable afterward.

## 2. Give it its own policy, with Action = Service Auth

This is the step that cost the most time.

Adding a `Service Token` include rule to the existing email-based Allow policy does nothing. Access still runs the identity login for anything under an `Allow` action, so the token gets ignored and the request still redirects.

The token needs a **separate policy** whose **Action** is set to **Service Auth**, not Allow. Under Include, one rule: `Service Token` equals the token created above. Leave the existing email policy untouched alongside it.

Two things that look like problems but are not:

- The **Policy Tester showing "blocked"** for your own email. The tester evaluates login identities only. It cannot simulate a request carrying service token headers, so a Service Auth policy will always show a human as blocked. That is correct behaviour.
- Creating the policy under **Access controls → Policies** makes a *reusable* policy, which does nothing until attached to an application. Check the application's Policies tab, or the "Used by applications" column in the policy list, to confirm it is actually bound.

## 3. n8n instance-level MCP token

Separate credential from the Cloudflare one, and separate from n8n's REST API key.

In n8n: **Settings → Instance-level MCP → Connection details → Access Token tab → generate**. Generating a new token revokes the previous one.

Also relevant: MCP clients can only see workflows explicitly enabled for MCP access. Instance-level MCP has to be on, and each workflow enabled individually via its workflow menu → Settings.

## 4. Verify with curl before touching any client config

Do not debug through a client. Prove the endpoint works first.

```
curl -X POST https://n8n.mallaegeking.org/mcp-server/http \
  -H "CF-Access-Client-Id: <id>" \
  -H "CF-Access-Client-Secret: <secret>" \
  -H "Authorization: Bearer <n8n mcp token>" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"curl-test","version":"1.0"}}}'
```

Success looks like an `event: message` block containing `"serverInfo":{"name":"n8n MCP Server"}`.

The POST body matters. A bare GET against the MCP endpoint hangs waiting for a handshake that never arrives, and Cloudflare eventually returns a `524` timeout. That is not a fault, just the wrong request shape.

Reading the intermediate failures, which are all informative:

| Response | Meaning |
|---|---|
| `302` to `cloudflareaccess.com/cdn-cgi/access/login/...` | Access is intercepting. Service token missing or policy wrong. |
| `Unauthorized: Authorization header not sent` | Past Cloudflare. n8n wants its own bearer token. |
| `Unauthorized` (bare) | Past Cloudflare, but the bearer token is wrong. Likely the REST API key instead of the MCP token. |
| `524` | Endpoint reached, no response in time. Usually a GET where a POST handshake was needed. |
| `502` | Cloudflare reached the tunnel, tunnel could not get a valid response from the origin. |

`curl` does not follow redirects by default, which is why the first case shows as a bare `302 Found` HTML stub. Add `-v` and grep for `location` to see where it actually goes.

## 5. Register the server in Claude Code

Single line, real values substituted, user scope:

```
claude mcp add -s user -t http n8n https://n8n.mallaegeking.org/mcp-server/http \
  -H "Authorization: Bearer <n8n mcp token>" \
  -H "CF-Access-Client-Id: <id>" \
  -H "CF-Access-Client-Secret: <secret>"
```

`-s user` matters. The default scope is `local`, which is per-directory, so a server added that way only appears when Claude Code is started from that exact folder. That silently looks identical to "the config did not save".

Confirm the write landed:

```
grep -n -A10 '"n8n"' ~/.claude.json
```

A correct entry has `type`, `url`, and a `headers` object with all three headers. An entry with only `type` and `url` is a leftover from the OAuth attempt and will always fail against Access.

Working result from `/mcp`:

```
Status:       ✔ connected
Auth:         ✔ authenticated
Capabilities: tools · resources
Tools:        34 tools
```

## Two config files, two Claude installs

A lot of wasted effort came from editing the wrong file.

Claude Desktop on Windows and Claude Code in WSL are separate installs with separate configs:

- Claude Desktop: `C:\Users\ticha\.claude.json`
- Claude Code in WSL: `/home/ticha/.claude.json`

Editing one has no effect on the other. The failure output names the file it is reading under `Config location:`. Trust that over assumption.

Account-level connectors added through claude.ai do sync across all clients automatically, which is what creates the expectation that everything else should too. Locally-defined servers do not.

Related: a stdio-mode server whose path is `C:\Users\...` cannot run under WSL. It would need `/mnt/c/Users/...`, or a fresh install inside the Linux filesystem.

## cloudflared was pinned to `latest` and never updated

`cloudflared.yaml` uses `image: cloudflare/cloudflared:latest` with `--no-autoupdate`. Those two together mean the pod runs whatever `latest` resolved to on the day it first started, forever. This instance sat on `2026.7.3` for 22 days while the logs warned daily about `2026.8.3`.

There is no `cloudflared` package on the host to upgrade. `apt-get install cloudflared` fails with "Unable to locate package" because it only exists as a container image in the cluster.

To pull the current image:

```
kubectl rollout restart deployment/cloudflared -n n8n
```

Confirm with `kubectl get pods -n n8n` and check the pod age and hash changed.

**Change to make:** pin an explicit version tag in `cloudflared.yaml` instead of `latest`. With GitOps, upgrades should be a visible commit, not a side effect of whenever a pod happens to restart.

## Diagnosing 502s: isolate the origin from the network path

Before the restart, the tunnel logs showed stream cancellations specifically on the MCP endpoint:

```
ERR error="stream 19865 canceled by remote with error code 0"
    originService=http://n8n.n8n.svc.cluster.local:80
ERR Request failed dest=https://n8n.mallaegeking.org/mcp-server/http
```

The single most useful test here is a port-forward, because it removes Cloudflare from the picture entirely:

```
kubectl port-forward -n n8n svc/n8n 8080:80
```

Then run the same POST handshake against `http://localhost:8080/mcp-server/http`, dropping both CF headers and keeping the bearer token. A successful handshake there proves n8n is healthy and puts the fault definitively in the network path.

Note the port-forward output says `Forwarding from 127.0.0.1:8080 -> 5678`. Port 80 is the Service, 5678 is the container. That mapping is from `service.yaml`.

If stream cancellations return, the next lever is **Disable Chunked Encoding**. Because this tunnel is remotely managed, that setting lives in the dashboard and not in any manifest or ConfigMap: Zero Trust → Networks → Tunnels → `homelab-k3s` → the n8n published application → Additional application settings → HTTP Settings.

## Credential inventory

Four distinct secrets are in play, and confusing them produces misleading errors:

| Credential | Where from | Used for |
|---|---|---|
| `CF-Access-Client-Id` / `-Secret` | Zero Trust → Service Auth | Getting past Cloudflare Access |
| n8n MCP access token | n8n Settings → Instance-level MCP | Authenticating to the MCP server |
| n8n REST API key | n8n Settings → API | Only needed by third-party tools hitting the REST API |
| `N8N_ENCRYPTION_KEY` | `openssl rand -hex 24` at setup | Encrypting stored credentials, never changes |

The n8n API key is not interchangeable with the MCP token. Using the wrong one returns a bare `Unauthorized` with no explanation.

## Update to the tunnel doc

`n8n-cloudflare-tunnel-setup.md` closes with a known limitation about external webhooks being blocked by the Access login gate, and suggests a bypass policy.

A service token is the better answer for that too. A bypass policy opens the path to anyone who knows the URL. A service token authenticates the caller, so the gate stays closed to everything else. Same mechanism as documented above, just a different client sending the headers.
