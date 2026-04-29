---
name: openfaas-function-dev
description: Develops OpenFaaS serverless functions in Python, Node.js, or Go using faas-cli. Use when scaffolding from templates, writing handlers, configuring stack.yaml, adding dependencies or secrets, building images, deploying to OpenFaaS, or iterating locally with local-run.
---

# OpenFaaS Function Development

## Pre-flight checks

Before generating any function code, verify the toolchain is available and configured.

1. Check `faas-cli` is installed: `faas-cli version`. If missing, install with `arkade get faas-cli` or `curl -sSL https://cli.openfaas.com | sudo sh`.
2. Inspect the output of `faas-cli version` for the gateway's orchestration provider. If it reports a Kubernetes server version, the cluster is Kubernetes-backed and `kubectl` is available as a break-glass option (see [Use kubectl only as a break glass](#use-kubectl-only-as-a-break-glass)).
3. Confirm Docker is running: `docker info`.
4. If deploying to a cluster, expect `OPENFAAS_URL` and `OPENFAAS_PREFIX` to be set (the latter is the image registry prefix used by `faas-cli new`, e.g. `ghcr.io/acme` or `ttl.sh/<user>`).

## Core workflow

The standard inner loop is:

```bash
faas-cli new --lang <template> <fn-name>     # scaffold
# edit ./<fn-name>/handler.<ext>
faas-cli local-run <fn-name>                  # iterate locally with Docker
# OR full cycle:
faas-cli up -f <fn-name>.yml                  # build + push + deploy
```

See [Testing with `local-run`](#testing-with-local-run) below for the local loop and [Iterating fast](#iterating-fast) for live-reload options.

## Testing with `local-run`

`faas-cli local-run` builds the function and starts it as a Docker container that listens on `http://127.0.0.1:8080`. **It is a long-running, foreground process** — it does not return until you stop it (Ctrl+C). Do not pipe input to it or expect it to exit on its own.

The workflow is two steps:

1. Start the function in a separate process — either in another terminal, in a tmux pane, or as a background process:

   ```bash
   faas-cli local-run --build my-fn &           # background, same shell
   # or run in another terminal:
   faas-cli local-run --build my-fn
   ```

2. Once you see `Listening on port: 8080`, invoke the function from elsewhere:

   ```bash
   curl -i http://127.0.0.1:8080 -d "hello"
   ```

When done, stop the container with Ctrl+C (foreground) or `kill %1` (background).

If port 8080 is already in use, override it with `--port`:

```bash
faas-cli local-run --build my-fn --port 3001
```

If `stack.yaml` has multiple functions, pass the function name as an argument: `faas-cli local-run --build <fn-name>`.

## Choosing a template

List available templates: `faas-cli template store list`. Recommended defaults:

| Language | Template | Notes |
|---|---|---|
| Python | `python3-http` | Flask-based HTTP. Default. |
| Python (native deps) | `python3-http-debian` | Debian base, supports `build_options` like `libpq`. |
| Node.js | `node22` | Express-based, async/await handler. |
| Go | `golang-middleware` | Uses `http.HandleFunc` from stdlib, supports modules and vendoring. |

Pull a template explicitly with `faas-cli template store pull <name>` if not auto-fetched.

## Scaffolding a function

```bash
export OPENFAAS_PREFIX=ttl.sh/<user>          # or your registry prefix
faas-cli new --lang python3-http hello-python
```

This creates:
- `hello-python/handler.py` (or `.js`, `.go`, etc.) — your code
- `hello-python/requirements.txt` (or `package.json`, `go.mod`) — dependencies
- `hello-python.yml` — the stack file

To append additional functions to the same stack file, use `--append`:

```bash
faas-cli new --lang node22 echo --append hello-python.yml
```

## Handler shapes per language

See [reference/handlers.md](reference/handlers.md) for complete handler templates and examples per language (Python, Node, Go).

Quick reference:

**Python (`python3-http`)** — return a dict with `statusCode`, `body`, optional `headers`. `event` exposes `body`, `headers`, `method`, `query`, `path`.

**Node.js (`node22`)** — `module.exports = async (event, context) => context.status(200).succeed(result)`. Async/await supported.

**Go (`golang-middleware`)** — `func Handle(w http.ResponseWriter, r *http.Request)` — write to `w`, read from `r`. Always `defer r.Body.Close()` if reading.

## stack.yaml essentials

Authoritative reference: [reference/stack-yaml.md](reference/stack-yaml.md). Common fields:

```yaml
version: 1.0
provider:
  name: openfaas
  gateway: http://127.0.0.1:8080
functions:
  my-fn:
    lang: python3-http
    handler: ./my-fn
    image: ${REGISTRY:-ttl.sh/myuser}/my-fn:${TAG:-latest}
    environment:
      LOG_LEVEL: info
    secrets:
      - my-fn-token
    build_options:
      - libpq                   # named bundle from template.yml
    build_args:
      ADDITIONAL_PACKAGE: "jq"  # extra apk/apt packages
    limits:
      memory: 128Mi
      cpu: 100m
    requests:
      memory: 64Mi
      cpu: 50m
    annotations:
      topic: kafka.payments-received
configuration:
  templates:
    - name: python3-http
      source: https://github.com/openfaas/python-flask-template
```

Notes:
- `image` supports `${VAR:-default}` envsubst — keep registry overridable.
- `handler` is always a folder, never a filename.
- Function name must be a DNS label (lowercase alphanumeric and `-`, max 63 chars).
- Pin templates by appending `#tag-or-branch` to the `source` URL for reproducible builds.
- `configuration.copy` lists project sub-paths to inject into every function's handler folder (for shared `./common`, `./models`, etc.).

## Dependencies

- **Python**: list packages in `<fn>/requirements.txt`. For native libs (e.g. `psycopg2`), use `python3-http-debian` and add `build_options: [libpq]` or `build_args: { ADDITIONAL_PACKAGE: "libpq-dev gcc python3-dev" }`.
- **Node**: `cd <fn> && npm install --save <pkg>`. Tests in `<fn>/test/` run automatically during `faas-cli build`.
- **Go**: `cd <fn> && go get <module>`. For private modules, set `GOPRIVATE=...`, run `go mod vendor`, commit `vendor/`. For sub-packages, add `replace handler/function => ./` to `go.mod`.

## Secrets

Always read secrets from files, never environment variables:

1. Create the secret: `faas-cli secret create my-token --from-file my-token.txt`. Prefer `faas-cli` for routine work; only fall back to `kubectl create secret` as a break glass when `faas-cli` cannot do the job and `faas-cli version` confirms a Kubernetes server (see [Use kubectl only as a break glass](#use-kubectl-only-as-a-break-glass)).
2. Bind in `stack.yaml`:
   ```yaml
   secrets:
     - my-token
   ```
3. Read at runtime from `/var/openfaas/secrets/my-token`.

For `local-run`, place the secret file at `./.secrets/my-token` so it can be bind-mounted.

Update with `faas-cli secret update`. Re-read the file on each invocation (or use `fsnotify`) since the kubelet rotates the mounted file.

## Build/deploy commands

| Command | Purpose |
|---|---|
| `faas-cli build -f stack.yml` | Build image into local Docker library |
| `faas-cli push -f stack.yml` | Push image to registry |
| `faas-cli deploy -f stack.yml` | Deploy to cluster |
| `faas-cli up -f stack.yml` | Build + push + deploy in one step |
| `faas-cli local-run [name]` | Build + run as a local Docker container on `:8080` |
| `faas-cli publish --platforms linux/arm64,linux/amd64` | Multi-arch build + push (use instead of build/push for ARM) |
| `faas-cli build --shrinkwrap` | Emit a `./build/<fn>/Dockerfile` for use with external builders |
| `faas-cli generate -f stack.yml` | Convert stack.yml to Kubernetes CRD YAML for GitOps |
| `faas-cli list -v` / `faas-cli describe <fn>` | Inspect deployed functions |
| `faas-cli logs <fn>` | Tail function logs |
| `echo "data" \| faas-cli invoke <fn>` | Invoke a deployed function |

Do not pass `--skip-push` when deploying to a Kubernetes cluster — the cluster pulls from a registry, so skipping the push leaves it with a missing or stale image. Only use `--skip-push` for faasd on the same host or when you have loaded the image into the cluster yourself (`kind load docker-image`, etc.).

## Always advance the image tag on deploy

**Critical:** Kubernetes will not pull a new image if the tag has not changed. Depending on the cluster's `imagePullPolicy` (commonly `IfNotPresent`), pushing a new image to the same tag — including `:latest` — can leave the old image running in the function pod. Every deploy must use a new, unique image tag.

Two ways to ensure tags advance:

1. **Let the CLI generate a unique tag** by passing `--tag` to `build`, `push`, `publish`, `deploy`, or `up`:

   | Value | Tag suffix | When to use |
   |---|---|---|
   | `--tag=digest` | content hash of the handler folder | Rebuilds and watch loops; new tag only when code actually changes. Recommended for `--watch`. |
   | `--tag=sha` | short Git commit SHA | CI/CD on a clean working tree. |
   | `--tag=branch` | branch + short SHA | Preview deploys per branch. |
   | `--tag=describe` | output of `git describe` | Release builds with annotated tags. |

   The generated value is appended to whatever tag is in `stack.yaml` (or `latest` if none). Example:

   ```bash
   faas-cli up -f stack.yml --tag=digest
   # builds and deploys image: ghcr.io/acme/my-fn:latest-abc123…
   ```

2. **Bump the tag manually in `stack.yaml`** using `envsubst`:

   ```yaml
   image: ghcr.io/acme/my-fn:${TAG:-0.1.0}
   ```

   ```bash
   TAG=0.1.1 faas-cli up
   ```

Avoid relying on `:latest` for cluster deploys. Reserve `:latest` for `local-run` only. The `--tag` flag combines with environment-variable substitution, so a CI pipeline can set a base version in `stack.yaml` and append `--tag=sha` for traceability.

## Iterating fast

- Use `faas-cli local-run --watch` for the tightest loop on a single function (no cluster needed).
- Use `faas-cli up --watch --tag=digest` when functions need cluster services (other functions, gateway). The `--tag=digest` is required here so each save produces a unique tag the cluster will actually pull.
- Use `ttl.sh/<user>` as registry for throwaway images during prototyping. **Warning: ttl.sh is a public, anonymous registry** — anyone who guesses the image path can pull it. Never use it for proprietary or customer code, secrets baked into images, or anything you would not publish openly. For private workloads use a private registry (GHCR private, ECR, GCR, Docker Hub private repo, Harbor, etc.) and run `faas-cli registry-login` to authenticate.

## Use kubectl only as a break glass

Drive function development, deployment, and inspection through `faas-cli`. Do not reach for `kubectl` for routine tasks like deploying, listing, invoking, scaling, fetching logs, or managing secrets — `faas-cli` covers these consistently across providers (Kubernetes, faasd, etc.).

Only use `kubectl` when **both** of these are true:

1. `faas-cli version` reports a Kubernetes server (the gateway is running on Kubernetes). On non-Kubernetes providers like faasd, `kubectl` is not applicable — do not use it.
2. The task is a genuine break-glass or deeper-debugging scenario that `faas-cli` cannot handle, for example:
   - Inspecting pod-level state: `kubectl -n openfaas-fn describe pod <fn>-<hash>`, `kubectl -n openfaas-fn get events`.
   - Diagnosing image pull, scheduling, or CrashLoopBackOff issues that `faas-cli logs` and `faas-cli describe` cannot explain.
   - Examining the OpenFaaS control plane in `openfaas` namespace (gateway, queue-worker, operator).
   - Checking node, RBAC, NetworkPolicy, or admission-controller behaviour interfering with a function.

When you do use `kubectl`, prefer read-only commands (`get`, `describe`, `logs`, `events`). Avoid mutating function resources (`apply`, `edit`, `delete`, `patch`) — make those changes through `stack.yaml` and `faas-cli` so the source of truth stays consistent. Note your reason for breaking glass in the response so the user understands why `faas-cli` was insufficient.

## Verification checklist

After scaffolding/editing:
1. `faas-cli build -f stack.yml` succeeds.
2. `faas-cli local-run <fn>` starts; `curl http://127.0.0.1:8080` returns expected response.
3. Handler returns proper status codes and `Content-Type` headers when returning JSON.
4. Secrets are read from `/var/openfaas/secrets/<name>`, not env vars.
5. `image:` field includes a registry prefix (not bare `<fn>:latest`) before pushing.
6. When deploying to a cluster, the image tag has advanced since the previous deploy — either via `--tag=digest`/`--tag=sha` or by bumping the tag in `stack.yaml`. Never re-deploy with an unchanged tag.

## When to load deeper references

- For full handler examples per language → read [reference/handlers.md](reference/handlers.md).
- For the complete stack.yaml schema and advanced fields → read [reference/stack-yaml.md](reference/stack-yaml.md).
- Official docs: https://docs.openfaas.com/languages/overview/ and blog: https://www.openfaas.com/blog/.
