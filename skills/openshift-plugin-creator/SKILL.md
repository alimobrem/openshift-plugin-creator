---
name: openshift-plugin-creator
description: Create a fully working OpenShift Console dynamic plugin for an operator. Use this skill whenever a user wants to build a console plugin, create a dynamic plugin, add a UI for their operator, scaffold an OpenShift plugin, or mentions console-plugin-template. Also trigger when users mention building views, dashboards, or management pages for their OpenShift operator's custom resources, even if they don't say "plugin" explicitly.
---

# OpenShift Console Dynamic Plugin Creator

Build a complete, production-ready OpenShift Console dynamic plugin for any operator. This skill analyzes the operator's GitHub repo, proposes an architecture, and generates a fully working plugin with real views, CI, tests, and deployment infrastructure.

**SDK:** `@openshift-console/dynamic-plugin-sdk`
**Template:** `openshift/console-plugin-template`
**UI:** PatternFly 6 with unified design token system (semantic tokens, dark mode compatible)

## Prerequisites Check

Before starting, verify the user has these tools available. Run the checks and report what's missing:

```bash
gh --version        # GitHub CLI, authenticated
oc version          # OpenShift CLI
node --version      # Node.js
yarn --version      # Yarn 4.13.0+
docker --version || podman --version  # Container runtime
```

The user also needs:
- A **GitHub account** with `gh auth login` completed — required for repo creation, CI, and sharing the plugin with others
- An **existing operator** with a public GitHub repo
- **OpenShift cluster access** — `oc` CLI installed, able to `oc login` (required)

**Optional (ask the user):**
- **Quay.io account** — for hosting the container image externally. If unavailable, use OpenShift's internal registry instead (see Phase 5).

If Node.js, Yarn, or Docker/Podman are missing, install them for the user rather than asking them to do it manually. Use the appropriate package manager for their platform (e.g., `brew install node` on macOS, `dnf install` on Fedora/RHEL). Install Yarn via `corepack enable && corepack prepare yarn@stable --activate`. These are standard dev tools with no configuration decisions — just install and move on.

Track the user's setup as a variable for later phases:
- `has_quay`: whether the user has a Quay.io account

---

## Phase 1: Discovery

### Step 1: Collect Basics

Ask these questions one at a time:

1. "What's the GitHub repo URL for the operator you're building this plugin for?"
2. "What GitHub org or user should the new plugin repo live under?"

### Step 2: Analyze the Operator

Fetch the operator's GitHub repo and read these files (use WebFetch or clone):

- **CRD definitions** — look in `api/`, `config/crd/`, `deploy/crds/`, `pkg/apis/`, or similar directories. Read the Go types or YAML manifests to understand every custom resource's spec and status fields, validation rules, and relationships.
- **Controllers** — look in `controllers/`, `pkg/controller/`, or `internal/controller/`. Understand what reconciliation loops exist and what resources are watched.
- **RBAC** — look in `config/rbac/` or `deploy/`. The permissions hint at which resources matter most.
- **README and docs** — understand the operator's purpose, primary user workflows, and intended audience.
- **Any existing CLI or UI tooling** — understand current user experience gaps the plugin should fill.

### Step 3: Suggest Plugin Name

Based on the operator name, suggest: `console-plugin-<operator-short-name>`

This name is used for both the GitHub repo and the plugin itself. Present it as: "Based on the operator, I'd suggest `console-plugin-<name>`. This will be both the plugin name and the repo name. Sound good, or would you prefer something different?"

### Step 4: Present Architecture Proposal

Synthesize the analysis into a concrete proposal. For each custom resource, state:
- What views to build (list, detail, create/edit form)
- Which CRD fields appear as table columns vs detail page fields vs form inputs
- Status indicators and how conditions map to visual states

#### Perspective Decision

Decide where the plugin lives in the console:

**Suggest a new perspective when:**
- The operator manages 3+ custom resources forming a cohesive domain
- Workflows are self-contained — users would spend most of their session within this plugin
- There's a distinct user persona separate from "cluster admin"

**Suggest adding to existing admin/dev nav when:**
- 1-2 custom resources
- Short workflows — create, check status, occasionally edit
- Resources relate to existing console concepts (networking, storage, compute)

**Suggest both when:**
- Complex enough for a dedicated perspective, but some resources also belong in admin nav for cluster admins

Explain your reasoning. For example: "Your operator manages 5 CRDs with interconnected workflows. I'd suggest a dedicated perspective so users have a focused experience — but I'd also add a summary card to the admin dashboard."

#### Proposal Format

Present the proposal clearly:

```
Plugin: console-plugin-<name>
Placement: [Admin nav / New perspective / Both]
Navigation structure:
  - Section: <section-name>
    - <ResourceA> (list + detail + create/edit)
    - <ResourceB> (list + detail)
  - Section: <section-name>
    - <ResourceC> (list + detail + create/edit)

Additional features:
  - Dashboard card: [yes/no — why]
  - Actions on core resources: [yes/no — which]
  - Status indicators: [describe]
```

Ask the user:
- "Do these views cover what you need?"
- "Any workflows missing?"
- "Any views you'd remove or add?"
- "Does the nav placement make sense?"

Iterate until the user approves. Then lock the architecture.

---

## Phase 2: Scaffold

Create the project and configure the skeleton.

### Steps

1. Create the repo from the template:
   ```bash
   gh repo create <org>/<plugin-name> --template openshift/console-plugin-template --public --clone
   cd <plugin-name>
   ```

2. Update `package.json` — set these fields in the `consolePlugin` section:
   - `name`: the plugin name
   - `displayName`: human-readable name derived from the operator
   - `description`: one-line description of what the plugin does
   - `exposedModules`: an entry for every page/component that will be referenced from `console-extensions.json`. Map module names to source paths (e.g., `"NodeHealthCheckListPage": "./src/components/NodeHealthCheckListPage"`)

3. Update `console-extensions.json` — add all extensions from the approved architecture:
   - `console.page/route` for each page
   - `console.navigation/href` for each nav item
   - `console.navigation/section` if grouping multiple items
   - `console.perspective` if a custom perspective was chosen
   - `console.dashboards/card` if a dashboard card was approved
   - Use `$codeRef` references matching the `exposedModules` keys
   - Use i18n-wrapped strings: `%plugin__<plugin-name>~Display Text%`

4. Update `i18next-parser.config.js` — set `defaultNamespace` to `plugin__<plugin-name>`

5. Update locale namespace in any existing translation files

6. Verify the skeleton builds:
   ```bash
   yarn install
   yarn build
   ```

7. Commit: "scaffold: configure plugin metadata and extensions"

### Checkpoint

Tell the user: "The project skeleton is ready — all routes and nav items are wired up with placeholder pages. Want to review the structure before I build out the actual views?"

---

## Phase 3: Build

Generate real, working views and components. Read `references/patternfly6.md` for PatternFly 6 component patterns and `references/sdk-guide.md` for SDK hooks and APIs before writing components.

### TypeScript Types

First, generate TypeScript interfaces from the CRD schemas. Create a `src/types/` directory with a file per resource. Each type should include:
- The full spec interface with all fields
- The status interface including conditions
- A `K8sResourceKind` wrapper using the SDK's types
- The `GroupVersionKind` and model definition for SDK hooks

### Per-Resource Components

For each custom resource in the approved plan, generate these components in `src/components/<ResourceName>/`:

**List Page** (`<Resource>ListPage.tsx`):
- Use `useK8sWatchResource` to watch the resource
- Use `useActiveNamespace` for namespace scoping
- Composable `Table` from `@patternfly/react-table` with `Thead`/`Tbody`/`Tr`/`Th`/`Td`
- Sortable columns derived from the CRD's key spec/status fields
- Toolbar with filtering (by name at minimum, by status if applicable)
- Status column using PatternFly `Label` with color mapped to conditions:
  - Green/success for healthy/ready states
  - Yellow/warning for degraded/progressing states
  - Red/danger for error/failed states
- Row actions dropdown: Edit, Delete
- Empty state with `EmptyState` component when no resources exist, with a "Create" primary action

**Detail Page** (`<Resource>DetailsPage.tsx`):
- `PageSection` layout with `Tabs`:
  - **Overview tab**: `DescriptionList` with key spec fields, conditions table showing all `.status.conditions`
  - **YAML tab**: SDK's `ResourceYAMLEditor` for direct YAML editing
  - Additional tabs for complex resources (e.g., Events, Related Resources)
- Page header with resource name, namespace, status badge, and action dropdown (Edit, Delete)

**Create/Edit Form** (`<Resource>Form.tsx`):
- PatternFly 6 `Form` with `FormGroup` for each spec field
- Field type mapping from the CRD schema:
  - `string` → `TextInput`
  - `enum` → `Select` with `SelectOption`
  - `boolean` → `Switch`
  - `integer` → `NumberInput`
  - `object` → nested `FormFieldGroup` with expandable section
  - `array of strings` → `TextInput` with add/remove buttons
  - `array of objects` → repeatable `FormFieldGroup`
- Required field marking from CRD validation
- Form-level validation before submit
- Uses `k8sCreate` for new resources, `k8sUpdate` for edits
- Success: navigate to the detail page
- Error: display inline `Alert` with the API error

**Delete Modal** (`<Resource>DeleteModal.tsx`):
- PatternFly `Modal` with danger variant
- Confirm resource name before deletion
- Uses `k8sDelete`
- Success: navigate back to list, show success alert

### Operator-Level Features

If approved in the architecture:

**Dashboard Card:**
- Extension type: `console.dashboards/card`
- Show aggregate status of the operator's resources (count by status)
- Use `useK8sWatchResource` to watch all instances
- PatternFly `Card` with status summary

**Actions on Core Resources:**
- Extension type: `console.action/resource-provider`
- Add actions to existing resource types (e.g., "Create NodeHealthCheck" on Node detail pages)

### Code Standards (Non-Negotiable)

Every component must follow these rules:

- **PatternFly 6 tokens only** — no hex colors, no rgb values, no hardcoded spacing. Use `var(--pf-t--global--...)` in CSS. The stylelint config in the template enforces this.
- **i18n every user-visible string** — `useTranslation('plugin__<plugin-name>')` in every component, wrap all display text with `t()`
- **Plugin-namespaced CSS classes** — prefix all custom classes with `<plugin-name>__` (e.g., `console-plugin-nhc__list-page`). Never use `.pf-` or `.co-` prefixes.
- **Functional components with hooks** — no class components
- **TypeScript strict** — proper typing, no `any`
- **No naked element selectors in CSS** — only class selectors to prevent console breakage

Commit after completing all views: "feat: implement all resource views and components"

---

## Phase 4: Infrastructure

Read `references/deployment.md` for detailed configuration patterns.

### CI — GitHub Actions

Create `.github/workflows/` with these workflow files:

**ci.yml** (on PR and push to main):
```yaml
- yarn install --immutable
- yarn lint
- yarn build
```

**build-and-push.yml** (on push to main, only if `has_quay` — skip this workflow otherwise):
```yaml
- Build Docker image
- Tag with commit SHA and 'latest'
- Login to Quay.io using secrets.QUAY_USERNAME and secrets.QUAY_PASSWORD
- Push to quay.io/<org>/<plugin-name>
```

If `has_quay`, tell the user they need to add `QUAY_USERNAME` and `QUAY_PASSWORD` as GitHub repo secrets. Walk them through: GitHub repo > Settings > Secrets and variables > Actions > New repository secret.

**e2e.yml** (manual trigger via workflow_dispatch):
```yaml
- yarn install --immutable
- Run Cypress tests headlessly
- Upload screenshots/videos as artifacts
```

### Helm Chart

Update the chart in `charts/openshift-console-plugin/`:
- `Chart.yaml`: set `name` and `description`
- `values.yaml`:
  - If `has_quay`: set `plugin.image` to `quay.io/<org>/<plugin-name>`
  - If no Quay: set `plugin.image` to `image-registry.openshift-image-registry.svc:5000/<namespace>/<plugin-name>`
- Configure port (9443), replicas (2)
- Verify `consoleplugin.yaml` template references the correct display name

### Tests

Generate Cypress E2E tests in `integration-tests/`:
- Test that each list page loads and renders the table
- Test that the create form opens and has the expected fields
- Test that the detail page renders tabs correctly
- Create test fixtures in `integration-tests/fixtures/` with sample CR YAML based on the CRD schema

Commit: "ci: add GitHub Actions, configure Helm chart, and add E2E tests"

---

## Phase 5: Ship

### Local Development

Walk the user through running the plugin locally:

```bash
# Terminal 1: Start the plugin dev server
yarn install
yarn start
# Plugin runs on http://localhost:9001

# Terminal 2: Connect to your cluster and start the console
oc login <your-cluster-url>
yarn start-console
# Console runs on http://localhost:9000
```

Tell them to open http://localhost:9000 and navigate to their plugin's pages. Explain that component changes hot-reload, but `console-extensions.json` changes require restarting `yarn start`.

### Image Registry Setup

**If `has_quay` — Quay.io setup:**

Guide the user step by step:

1. Go to quay.io and create a new repository named `<plugin-name>` (or confirm it exists)
2. Create a robot account: User Settings > Robot Accounts > Create Robot Account
3. Grant the robot write access to the repository
4. Copy the robot credentials
5. Add them as GitHub secrets:
   - `QUAY_USERNAME`: the robot account name (format: `orgname+robotname`)
   - `QUAY_PASSWORD`: the robot token
6. Test the pipeline:
   ```bash
   docker build -t quay.io/<org>/<plugin-name>:test .
   docker push quay.io/<org>/<plugin-name>:test
   ```

**If no Quay — OpenShift internal registry:**

Build and push directly to the cluster's internal registry. No external account needed.

```bash
# Expose the internal registry (if not already exposed)
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge

# Get the registry route
REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')

# Login to the internal registry
docker login -u $(oc whoami) -p $(oc whoami -t) $REGISTRY

# Build and push
docker build -t $REGISTRY/<namespace>/<plugin-name>:latest .
docker push $REGISTRY/<namespace>/<plugin-name>:latest
```

The image is accessible within the cluster at `image-registry.openshift-image-registry.svc:5000/<namespace>/<plugin-name>:latest`.

### Cluster Deployment

Deploy to an OpenShift cluster:

**If `has_quay`:**
```bash
docker build -t quay.io/<org>/<plugin-name>:latest .
docker push quay.io/<org>/<plugin-name>:latest

helm upgrade -i <plugin-name> charts/openshift-console-plugin \
  -n <plugin-name> --create-namespace \
  --set plugin.image=quay.io/<org>/<plugin-name>:latest
```

**If no Quay (internal registry):**
```bash
# Build and push to internal registry (see above)
# Then deploy with Helm using the internal image path
helm upgrade -i <plugin-name> charts/openshift-console-plugin \
  -n <plugin-name> --create-namespace \
  --set plugin.image=image-registry.openshift-image-registry.svc:5000/<namespace>/<plugin-name>:latest
```

**Both paths — verify and enable:**
```bash
# Verify the ConsolePlugin CR was created
oc get consoleplugin <plugin-name>

# Enable the plugin
oc patch consoles.operator.openshift.io cluster \
  --patch '{"spec":{"plugins":["<plugin-name>"]}}' --type=merge

# Verify in the console — navigate to the plugin's pages
```

### Wrap-Up

Summarize what was created:
- GitHub repo URL
- Image location: Quay URL (if `has_quay`) or internal registry path
- List of all views and extensions
- CI pipeline status
- Deployed cluster (if applicable)

Suggest next steps:
- Adding more views or features
- Setting up a release/tagging workflow
- Contributing the plugin reference back to the operator repo
- Setting up OLM integration for automatic plugin discovery
