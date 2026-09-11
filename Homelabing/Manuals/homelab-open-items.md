# Homelab: Open Items

Everything left unfinished across the Proxmox template, k3s cluster, Argo CD, and n8n work. Roughly ordered by how much each one unblocks.

## Broken / needs fixing

**forge: database never migrated**
The deployed instance runs but throws `SQLITE_ERROR: no such table: programs` on login. The image was built and pushed manually, which skipped the schema migration step (the original `docker-compose.yml` had a separate "db-tools" target for this). Fix as part of the GitHub Actions work below: either an initContainer that runs the migration before the app container starts, or bake it into the image's startup command.

## Next build steps

**GitHub Actions CI for forge**
Automate the build-and-push that was done by hand: push code → image builds → pushes to ghcr.io → Argo CD picks up the new version. This is also where the migration fix lands. Directly matches the CI/CD track already on the learning plan.

**Sealed Secrets**
Currently every secret (forge's `APP_PASSCODE`, n8n's encryption key, the Cloudflare tunnel token) is created by hand with `kubectl` and lives nowhere in Git. That means a cluster rebuild requires recreating them manually from KeePass, and there's no record of which secrets an app needs. Sealed Secrets encrypts them so they can safely be committed alongside the manifests. Security-oriented, worth learning properly.

**SSH key rotation / SSH certificates**
Already logged in Todoist under Homelab Setup List. Two directions: a rotation script that updates `authorized_keys` and the Cloud-Init template field across hosts, or switching to SSH certificates via a private CA (`ssh-keygen -s`), where hosts trust the CA once and short-lived certs get issued after. The certificate approach scales much better past a handful of VMs.

**deutsch-tools on the cluster**
The other own-code project. Same pattern as forge once the CI piece is working.

## Smaller / cleanup

**Bake `qemu-guest-agent` into template 901**
Currently installed by hand on every clone. Two ways to do it without breaking the "never boot 901 directly" rule (both documented in the cloud-init template doc): boot it deliberately then reset machine-id before re-templating, or edit the disk offline with `virt-customize` (which also needs the machine-id reset, since installing a package triggers systemd to generate one).

**Enable the QEMU Guest Agent option on template 901**
`qm set 901 --agent enabled=1`. Installing the package inside the guest isn't enough on its own, Proxmox also needs to be told to expect it. Enables IP reporting in the summary panel and clean filesystem freeze/thaw during backups. One-time setting on the template.

**k3s-node-3 (VM 404)**
Cloned from the node template but never joined to the cluster. Spare capacity, ready whenever a third agent is wanted. Just needs the FritzBox reservation plus the agent install command.

**Cloudflare Access bypass for webhooks**
Once n8n workflows start using external webhooks, those paths need an Access bypass policy or the calling service gets blocked by the login gate.

## To learn / read up on

**DHCP reservations vs static leases vs static-on-host**
The controller uses a static IP configured in netplan inside the guest, the node clones use FritzBox reservations. Both work, they solve the problem from opposite ends. Worth understanding the tradeoff properly rather than picking whichever was already documented.

**Local DNS for the LAN**
`nip.io` and hosts-file entries both work but are workarounds. A Pi-hole or AdGuard Home instance would give proper local DNS records for the whole network, defined once and picked up by every device. Good future homelab addition.

**GitOps concepts, deeper**
Set aside during the hands-on session in favour of getting things working. Worth returning to: what happens on manual `kubectl edit` with self-heal on, app-of-apps patterns, sync waves.

## Planned, further out

**Self-hosted GitLab on Proxmox**
Everything built so far transfers: GitLab has its own container registry and CI system, and Argo CD doesn't care where the Git repo lives. Nothing learned here gets thrown away in the move.
