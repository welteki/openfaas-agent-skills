---
name: openfaas-function-dev
description: Develops OpenFaaS serverless functions in Python, Node.js, or Go using faas-cli. Use when scaffolding from templates, writing handlers, configuring stack.yaml, adding dependencies or secrets, building images, deploying to OpenFaaS, or iterating locally with local-run.
---

# OpenFaaS Function Development

Guides creation and iteration of OpenFaaS functions: scaffolding via templates, writing handlers in supported languages, configuring `stack.yaml`, managing secrets, and building/deploying with `faas-cli`.

## Pre-flight checks

Before generating any function code, verify the toolchain is available and configured.

1. Check `faas-cli` is installed: `faas-cli version`. If missing, install with `arkade get faas-cli` or `curl -sSL https://cli.openfaas.com | sudo sh`.
2. Confirm Docker is running: `docker info`.
3. If deploying to a cluster, expect `OPENFAAS_URL` and `OPENFAAS_PREFIX` to be set (the latter is the image registry prefix used by `faas-cli new`, e.g. `ghcr.io/acme` or `ttl.sh/<user>`).

## Core workflow

The standard inner loop is:

```bash
faas-cli new --lang <template> <fn-name>     # scaffold
# edit ./<fn-name>/handler.<ext>
faas-cli local-run <fn-name>                  # iterate locally with Docker
# OR full cycle:
faas-cli up -f <fn-name>.yml                  # build + push + deploy
```

Use `faas-cli up --watch --tag=digest` for live-reload during cluster-based development.
Use `faas-cli local-run --watch` for live-reload of a single function locally without a cluster.

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

1. Create the secret: `faas-cli secret create my-token --from-file my-token.txt` (or via `kubectl create secret`).
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
| `faas-cli up --skip-push` | Useful for KinD/local registries |
| `faas-cli local-run [name]` | Build + run as a local Docker container on `:8080` |
| `faas-cli publish --platforms linux/arm64,linux/amd64` | Multi-arch build + push (use instead of build/push for ARM) |
| `faas-cli build --shrinkwrap` | Emit a `./build/<fn>/Dockerfile` for use with external builders |
| `faas-cli generate -f stack.yml` | Convert stack.yml to Kubernetes CRD YAML for GitOps |
| `faas-cli list -v` / `faas-cli describe <fn>` | Inspect deployed functions |
| `faas-cli logs <fn>` | Tail function logs |
| `echo "data" \| faas-cli invoke <fn>` | Invoke a deployed function |

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
- Use `ttl.sh/<user>` as registry for throwaway images during prototyping.

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
