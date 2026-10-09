# Flogo Projects

TIBCO Flogo® applications. Each app lives under [`Apps/`](Apps/) as a self-contained `.flogo` file.

## Apps

### flogo-greeting

A simple REST service that returns a styled HTML **status page** for the running instance.
It is a Flogo re-implementation of a BusinessWorks 6 flow, using Flogo's built-in
resolvers in place of the original Java activity.

- **File:** [`Apps/flogo-greeting.flogo`](Apps/flogo-greeting.flogo)
- **Trigger:** REST (`tibco-wi-rest`) on port **9999**
- **Endpoint:** `GET /Greeting/{Name}`
- **Response:** `200 OK`, `Content-Type: text/html`

#### What it returns

An HTML card greeting `{Name}` followed by a table of runtime facts:

| Field | Source | Resolver |
|---|---|---|
| Host | Hostname of the running instance | `$env[HOSTNAME]` |
| Flogo Version | Flogo runtime version | `$property["FlogoVersion"]` app property |
| App Version | Application version | `$flowctx["AppVersion"]` |

These native resolvers replace the BusinessWorks 6 Java activity that originally
supplied the host name, BW version, and app version.

> **Note:** there is no runtime resolver for the Flogo engine version, so it is held
> in the `FlogoVersion` **application property** (currently `2.26.8`). Update that
> property if you build against a different Flogo version.

The `{Name}` path parameter is sanitised (strips `<`, `>`, `&`) before being embedded
in the page to avoid HTML injection.

#### Example

```bash
curl http://localhost:9999/Greeting/Dave
```

Returns an HTML page rendering:

```
Hello Dave
Host           <hostname>
Flogo Version  2.26.8
App Version    1.0.0
```

#### How the HTML + Content-Type is set

The REST reply uses the trigger's **`responseBody`** field (`{ body, headers }`) rather
than the plain `message` field, so a `Content-Type: text/html` header can be returned
alongside the HTML body. The body is built with `string.concat` and
`string.replaceAll` (for sanitisation).

## Building and running locally

Build an executable with the Flogo build CLI (`flogobuild`), using the build context
for your installed Flogo version:

```bash
flogobuild build-exe -f Apps/flogo-greeting.flogo -o ./bin -n flogo-greeting
```

Run it. Two environment settings are needed in a local (non-container) run:

```bash
# FLOGO_ENGINE_DELAY_LICENSE_CHECK bypasses the control-plane license check when offline.
# HOSTNAME must be set because the app uses the $env[HOSTNAME] resolver, which
# hard-fails at engine init if it is empty (containers/K8s always set it).
FLOGO_ENGINE_DELAY_LICENSE_CHECK=true HOSTNAME="$(hostname)" ./bin/flogo-greeting
```

Then browse to `http://localhost:9999/Greeting/<yourname>`.

## Deploying to the TIBCO Platform

Deploy to a dataplane with the TIBCO Platform CLI (`tibcop`). Outline:

1. `tibcop flogo:list-flogo-versions --dataplane-name <dp>` — pick a `buildtypeTag`.
2. `tibcop flogo:create-build --dataplane-name <dp> --flogo-version <tag> Apps/flogo-greeting.flogo`
3. `tibcop flogo:generate-values-from-build --dataplane-name <dp> --build-id <id> --output-dir Apps`
4. `tibcop flogo:deploy-app-release --dataplane-name <dp> --eula Apps/values.yaml`
5. `tibcop flogo:scale-app --dataplane-name <dp> --app-id <id> --count 1`

Use `deploy-app-release` (not `deploy-app`) so the required `--eula` flag is accepted.
Authenticate with your own TIBCO Platform profile/token — **never commit credentials**.

## Editing

Edit `.flogo` files with the Flogo VS Code extension, or from the command line with
the Flogo Design CLI (`flogodesign-cli` / `fda`). Validate mappings before building:

```bash
fda cm -f Apps/flogo-greeting.flogo
```
