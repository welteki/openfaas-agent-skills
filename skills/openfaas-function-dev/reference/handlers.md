# Handler Reference

Complete handler examples for the supported OpenFaaS templates: Python, Node.js, Go.

## Python (`python3-http` / `python3-http-debian`)

The handler receives `event` and `context`:

- `event.body` — request body (string for text, parsed dict for JSON content-type)
- `event.headers` — dict of HTTP headers
- `event.method` — HTTP method
- `event.query` — query parameters
- `event.path` — request path
- `context.hostname` — function hostname

Response is a dict:

```python
def handle(event, context):
    return {
        "statusCode": 200,
        "body": {"message": "Hello from OpenFaaS!"},
        "headers": {
            "Content-Type": "application/json"
        }
    }
```

### Reading a secret

```python
def read_secret(name):
    with open(f"/var/openfaas/secrets/{name}") as f:
        return f.read().strip()

def handle(event, context):
    token = read_secret("python-auth-token")
    if "Authorization" not in event.headers:
        return {"statusCode": 401, "body": "Unauthorized"}
    if event.headers["Authorization"] != f"Bearer {token}":
        return {"statusCode": 401, "body": "Unauthorized"}
    return {"statusCode": 200, "body": "Access granted"}
```

### Reading a static file bundled with the function

```python
import os, json

def handle(event, context):
    current_dir = os.path.dirname(os.path.abspath(__file__))
    with open(os.path.join(current_dir, "data.json")) as f:
        data = json.load(f)
    return {
        "statusCode": 200,
        "body": json.dumps(data),
        "headers": {"Content-Type": "application/json"}
    }
```

### Native deps (Postgres example)

`stack.yml`:

```yaml
functions:
  pgfn:
    lang: python3-http-debian
    handler: ./pgfn
    image: pgfn:latest
    build_options:
      - libpq
    secrets:
      - pg-conn
```

`requirements.txt`:

```
psycopg2==2.9.3
```

## Node.js (`node22`)

```js
'use strict'

module.exports = async (event, context) => {
  const result = {
    body: JSON.stringify(event.body),
    'content-type': event.headers['content-type']
  }
  return context.status(200).succeed(result)
}
```

### With an npm dependency

```bash
cd http-req
npm install --save axios
```

```js
'use strict'
const axios = require('axios')

module.exports = async (event, context) => {
  if (!event.body.url) {
    return context.status(400).fail('Missing .url in request body')
  }
  try {
    const res = await axios({ method: 'GET', url: event.body.url, validateStatus: () => true })
    return context.status(res.status).succeed(res.data)
  } catch (e) {
    return context.status(500).fail(e)
  }
}
```

### Reading a secret

```js
'use strict'
const fs = require('fs').promises

const tokenSecretName = 'node-fn-token'

module.exports = async (event, context) => {
  const token = (await fs.readFile(`/var/openfaas/secrets/${tokenSecretName}`, 'utf8')).trim()
  if (!event.headers.authorization) return context.status(401).fail('Unauthorized')
  if (event.headers.authorization !== `Bearer ${token}`) return context.status(403).fail('Forbidden')
  return context.status(200).succeed('Authenticated')
}
```

### Unit tests

`package.json`:

```json
{
  "scripts": { "test": "mocha test/test.js" },
  "devDependencies": { "chai": "^4.2.0", "mocha": "^7.0.1" }
}
```

Tests under `<fn>/test/` are run automatically during `faas-cli build`. Failed tests fail the build.

## Go (`golang-middleware`)

```go
package function

import (
    "fmt"
    "io"
    "net/http"
)

func Handle(w http.ResponseWriter, r *http.Request) {
    var input []byte
    if r.Body != nil {
        defer r.Body.Close()
        body, _ := io.ReadAll(r.Body)
        input = body
    }
    w.WriteHeader(http.StatusOK)
    w.Write([]byte(fmt.Sprintf("Body: %s", string(input))))
}
```

### Adding dependencies

```bash
cd bcrypt
go get golang.org/x/crypto/bcrypt
```

`go.mod` is updated automatically.

### Sub-packages

If you create `handlers/handlers.go` with its own `go.mod`, add to the function's `go.mod`:

```
replace handler/function => ./
```

Or run: `go mod edit -replace handler/function=./`.

### Private modules

```bash
cd secret-fn
GOPRIVATE=github.com/acmecorp go get
GOPRIVATE=github.com/acmecorp go mod vendor
```

Commit the `vendor/` directory.


