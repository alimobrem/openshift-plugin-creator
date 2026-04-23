# Deployment Reference

## GitHub Actions CI/CD

### ci.yml — Lint & Type Check

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Enable Corepack
        run: corepack enable

      - name: Install dependencies
        run: yarn install --immutable

      - name: Lint
        run: yarn lint

      - name: Build
        run: yarn build
```

### build-and-push.yml — Container Build & Quay Push

```yaml
name: Build and Push

on:
  push:
    branches: [main]

env:
  IMAGE_REGISTRY: quay.io
  IMAGE_REPOSITORY: ${{ github.repository_owner }}/PLUGIN_NAME
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_REPOSITORY }}:${{ env.IMAGE_TAG }} .

      - name: Tag latest
        run: docker tag ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_REPOSITORY }}:${{ env.IMAGE_TAG }} ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_REPOSITORY }}:latest

      - name: Login to Quay.io
        uses: docker/login-action@v3
        with:
          registry: ${{ env.IMAGE_REGISTRY }}
          username: ${{ secrets.QUAY_USERNAME }}
          password: ${{ secrets.QUAY_PASSWORD }}

      - name: Push image
        run: |
          docker push ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_REPOSITORY }}:${{ env.IMAGE_TAG }}
          docker push ${{ env.IMAGE_REGISTRY }}/${{ env.IMAGE_REPOSITORY }}:latest
```

Replace `PLUGIN_NAME` with the actual plugin name when generating.

### e2e.yml — Cypress E2E Tests

```yaml
name: E2E Tests

on:
  workflow_dispatch:

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Enable Corepack
        run: corepack enable

      - name: Install dependencies
        run: yarn install --immutable

      - name: Run Cypress
        run: yarn test-cypress-headless

      - name: Upload artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: cypress-results
          path: |
            integration-tests/screenshots
            integration-tests/videos
```

## Quay.io Setup

### Creating a Robot Account

1. Log in to quay.io
2. Navigate to your organization (or user account)
3. Go to **Robot Accounts** (gear icon > Robot Accounts)
4. Click **Create Robot Account**
5. Name it (e.g., `ci_push`)
6. Grant it **Write** permission to the plugin repository
7. Copy the credentials:
   - Username: `orgname+ci_push` (or `username+ci_push`)
   - Token: the generated token

### Adding GitHub Secrets

1. Go to the GitHub repository
2. Settings > Secrets and variables > Actions
3. Click **New repository secret**
4. Add `QUAY_USERNAME` with the robot account username
5. Add `QUAY_PASSWORD` with the robot account token

### Creating the Quay Repository

```bash
# Or create via the Quay.io web UI
# Navigate to quay.io > New Repository > Name it > Public
```

## Helm Deployment

### Install/Upgrade

```bash
helm upgrade -i <plugin-name> charts/openshift-console-plugin \
  -n <plugin-name> --create-namespace \
  --set plugin.image=quay.io/<org>/<plugin-name>:latest
```

### Verify Deployment

```bash
# Check the deployment
oc get deployment -n <plugin-name>

# Check the service
oc get service -n <plugin-name>

# Check the ConsolePlugin CR
oc get consoleplugin <plugin-name> -o yaml

# Check pods are running
oc get pods -n <plugin-name>
```

### Enable the Plugin

```bash
# Option 1: CLI
oc patch consoles.operator.openshift.io cluster \
  --patch '{"spec":{"plugins":["<plugin-name>"]}}' --type=merge

# Option 2: Console UI
# Administration > Cluster Settings > Configuration > Console operator
# Add the plugin name to the plugins list
```

### Verify Plugin is Active

```bash
# Check console operator config
oc get consoles.operator.openshift.io cluster -o jsonpath='{.spec.plugins}'

# Check plugin is loaded (in browser console)
# window.SERVER_FLAGS.consolePlugins should include your plugin
```

### Uninstall

```bash
# Remove from console plugins list
oc patch consoles.operator.openshift.io cluster \
  --patch '{"spec":{"plugins":[]}}' --type=merge

# Uninstall Helm release
helm uninstall <plugin-name> -n <plugin-name>

# Delete namespace
oc delete namespace <plugin-name>
```

## Local Development

### Quick Start

```bash
# Terminal 1: Plugin dev server
yarn install
yarn start
# Runs on http://localhost:9001

# Terminal 2: Console bridge
oc login <cluster-url>
yarn start-console
# Runs on http://localhost:9000
```

### What start-console Does

The `start-console.sh` script:
1. Reads plugin metadata from package.json
2. Pulls the `quay.io/openshift/origin-console:latest` container
3. Connects to the logged-in cluster using oc credentials
4. Exposes the console on port 9000
5. Registers the plugin from localhost:9001

### Hot Reload Behavior

- **Component changes**: Hot-reload automatically
- **CSS changes**: Hot-reload automatically
- **console-extensions.json changes**: Require restarting `yarn start`
- **package.json consolePlugin changes**: Require restarting `yarn start`

### Debugging

```bash
# Check if plugin is registered (browser console)
window.SERVER_FLAGS.consolePlugins

# Check plugin store (dev builds only)
window.pluginStore

# Disable a plugin for debugging
# Add ?disable-plugins=<plugin-name> to the URL
```

## ConsolePlugin CR Reference

```yaml
apiVersion: console.openshift.io/v1
kind: ConsolePlugin
metadata:
  name: <plugin-name>
spec:
  displayName: "Human-Readable Plugin Name"
  backend:
    type: Service
    service:
      name: <plugin-name>
      namespace: <plugin-namespace>
      port: 9443
      basePath: /
  i18n:
    loadType: Preload
  proxy:
    - alias: "api-proxy"
      endpoint:
        type: Service
        service:
          name: <backend-service>
          namespace: <namespace>
          port: 8080
      authorization: UserToken
```

### Proxy Configuration

If the plugin needs to talk to a backend service (e.g., the operator's API):

```yaml
proxy:
  - alias: "my-backend"
    endpoint:
      type: Service
      service:
        name: my-operator-service
        namespace: my-operator-namespace
        port: 8080
    authorization: UserToken  # Forward the user's token
```

Access from the plugin:
```typescript
const response = await consoleFetch('/api/proxy/plugin/<plugin-name>/my-backend/endpoint');
```

## Nginx Configuration

The template's ConfigMap sets up nginx to serve the plugin:

- Listens on configured port (default 9443) with TLS
- Certificates from `/var/cert/tls.crt` and `/var/cert/tls.key` (auto-generated by OpenShift)
- Serves static files from `/usr/share/nginx/html`
- Logs to stdout for container log aggregation
- IPv6 support enabled
