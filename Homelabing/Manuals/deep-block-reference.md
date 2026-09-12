# Deep block reference

> **Historical.** The session plan moved to the Notion "Threads" database (Track: Infra / AI & Automation). Kept for the reasoning behind the two-track structure. Do not add new sessions here.

These are a reference, not deadlines. Slower is fine, vague is not.

Two tracks, two sessions each per week:
- Infra runs Monday and Thursday.
- AI and automation runs Tuesday and Friday.

The spine of both is your own homelab and deutsch-tools, so you are building real things, not tutorials.

---

## Infra track (Docker and Kubernetes)

Spine: get deutsch-tools running properly on k3s, then layer GitOps, CI, Terraform, and monitoring on top.

### 1. Containerize deutsch-tools
Write a clean multi-stage Dockerfile for the Vite/React app. Build the image, run it with `docker run`, confirm it serves in the browser.
Done when: the image builds and the container serves the app locally.

### 2. Deploy to k3s by hand
Write raw manifests: Deployment, Service, Ingress. `kubectl apply` them. Reach the app through the ingress.
Done when: the app runs on k3s and is reachable, no Helm, no shortcuts.

### 3. Package with Kustomize
Split the manifests into a base and one overlay. Parametrize the image tag and replica count.
Done when: `kustomize build` renders and deploys cleanly.

### 4. GitOps with Argo CD
Install Argo CD on the cluster. Point an Application at your git repo. Push a change and watch it sync.
Done when: a commit to git updates the cluster with no manual kubectl.

### 5. CI on every commit
GitHub Actions workflow that builds the image, pushes it to a registry, and bumps the tag in your manifests repo.
Done when: pushing code triggers a build and publishes an image automatically.

### 6. Codify with Terraform
Move something real into Terraform: namespaces, the Argo CD install, or the app resources. Version it.
Done when: `terraform apply` provisions it reproducibly from an empty state.

### 7. Observability
Deploy Prometheus and Grafana on k3s. Scrape the app. Build one dashboard and wire one alert.
Done when: a Grafana dashboard shows live metrics and one alert fires on a test condition.

At this point the full loop exists: commit, CI builds, Argo deploys, Terraform holds the infra, Grafana watches it. That is a platform engineer's portfolio in one repo.

---

## AI and automation track (Claude, agents, n8n)

Spine: build one real automation end to end, growing from a trivial n8n flow into a Claude-powered agent that runs itself on the homelab.

### 1. n8n back online
Get n8n running (homelab or Docker Desktop). Build one trivial flow end to end: webhook in, transform, response out.
Done when: you trigger the flow and it returns a result.

### 2. Put Claude in the loop
Add an HTTP node that calls the Anthropic API. Feed it a German word, get back an example sentence and translation for deutsch-tools.
Done when: the flow returns real LLM output from your input.

### 3. Make it reliable
Force the model to return JSON. Parse it in n8n. Add error handling and a retry.
Done when: the flow produces clean structured data and survives a bad response without crashing.

### 4. Claude Code on a real task
Use Claude Code with your f-mode skill on an actual repo change. Read how MCP tools are exposed to it.
Done when: you ship one real code change through Claude Code, verified and committed.

### 5. First agent loop
Build a flow that takes a goal, calls a tool, and acts on the result. For example: Firecrawl scrapes a page, Claude summarizes it, the summary gets stored.
Done when: a multi-step agent runs one tool and produces a useful artifact.

### 6. Give the agent real data
Wire one integration into the agent: Todoist, Calendar, or Firecrawl through an MCP server or n8n node.
Done when: the agent reads from or writes to a real service.

### 7. Ship it self-hosted
Move your best workflow onto the homelab and put it on a schedule so it runs without you.
Done when: it runs on a timer, on your own infrastructure, hands off.

---

## The rule that makes this work

Open this file at the start of every deep block. Find the next unchecked session for that track. Do only that one. Close the block by committing or by writing one line on where you stopped. Next block, continue from there. The point is never to finish fast. The point is that the block is never spent deciding what to do.
