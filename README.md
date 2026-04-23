# OpenShift Plugin Creator

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that creates fully working OpenShift Console dynamic plugins for Kubernetes operators. Point it at your operator's GitHub repo and it builds a complete plugin — real views, CI/CD, Helm chart, and deployment — ready to run on your cluster.

## What It Does

Given an operator with custom resources, this skill:

1. **Analyzes** your operator's CRDs, controllers, and RBAC to understand its resources
2. **Proposes** a plugin architecture with ASCII mockups — which views to build, where they go in the console, and how they'll look before any code is written
3. **Scaffolds** a project from the official [console-plugin-template](https://github.com/openshift/console-plugin-template) with all routes and nav items wired up
4. **Builds** production-quality views for every custom resource — list pages, detail pages, create/edit forms, delete modals — using PatternFly 6 and the console SDK
5. **Adds infrastructure** — GitHub Actions CI, Helm chart, Cypress E2E tests
6. **Deploys** the plugin to your OpenShift cluster via Helm

Each phase has a checkpoint where you review and adjust before proceeding.

## What You Get

- TypeScript types generated from your CRD schemas
- List pages with sortable columns, filtering, and status badges
- Detail pages with overview, YAML editor, and conditions tabs
- Create/edit forms with field types mapped from your CRD spec
- Delete confirmation modals
- Dashboard cards and actions on core resources (when appropriate)
- GitHub Actions for lint, build, container push, and E2E tests
- Helm chart configured for your image registry
- Full i18n support and dark mode compatibility

## Installation

```bash
claude plugin marketplace add alimobrem/openshift-plugin-creator
claude plugin install openshift-plugin-creator
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [GitHub CLI](https://cli.github.com/) (`gh`) installed and authenticated
- [OpenShift CLI](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html) (`oc`) installed
- Access to an OpenShift cluster
- An existing operator with a public GitHub repo

Node.js, Yarn, and Docker/Podman are installed automatically if missing.

A [Quay.io](https://quay.io/) account is optional — without one, the skill uses OpenShift's internal image registry instead.

## Usage

Start Claude Code and describe what you want:

```
claude

> Build a console plugin for my operator at github.com/my-org/my-operator
```

Or invoke the skill directly:

```
> /openshift-plugin-creator
```

The skill walks you through each phase interactively. It asks two questions up front (your operator repo URL and where the new plugin repo should live), then analyzes the operator and presents an architecture proposal with ASCII mockups for your approval.

## Example

```
> I need a UI for my NodeHealthCheck operator at github.com/medik8s/node-healthcheck-operator

Analyzing the operator...

Found 2 CRDs:
  - NodeHealthCheck (spec: selector, remediationTemplate, minHealthy, ...)
  - NodeHealthCheckConfig (spec: maxUnhealthy)

Plugin: console-plugin-node-healthcheck
Placement: Admin nav under Compute
Navigation:
  - Section: Node Health
    - NodeHealthCheck (list + detail + create/edit)
    - NodeHealthCheckConfig (list + detail)

┌──────────────────────────────────────────────────────────┐
│ NodeHealthChecks                     [Create] [Filter ▾] │
├──────────────┬────────────┬──────────┬──────────────────┤
│ Name         │ Namespace  │ Status   │ Min Healthy      │
├──────────────┼────────────┼──────────┼──────────────────┤
│ nhc-workers  │ medik8s    │ ● Ready  │ 51%              │
│ nhc-infra    │ medik8s    │ ◐ Progressing │ 40%          │
│ nhc-all      │ default    │ ✖ Failed │ 75%              │
└──────────────┴────────────┴──────────┴──────────────────┘

Do these views and layouts look right?
```

## How It Works

The skill uses reference documents for SDK APIs, PatternFly 6 patterns, template structure, and deployment configuration. All generated code follows these standards:

- **PatternFly 6 design tokens** — no hardcoded colors or spacing
- **Console SDK hooks** — `useK8sWatchResource`, `k8sCreate`, `k8sUpdate`, `k8sDelete`
- **Plugin-namespaced CSS** — prevents style conflicts with the console
- **Strict TypeScript** — proper typing, no `any`
- **i18n** — every user-visible string wrapped with `t()`

## Project Structure

```
skills/openshift-plugin-creator/
  SKILL.md                         # Main skill (all phases)
  references/
    sdk-guide.md                   # SDK extension types, hooks, APIs
    patternfly6.md                 # PF6 components, tokens, imports
    template-structure.md          # console-plugin-template file reference
    deployment.md                  # Helm, CI, registry, cluster deployment
```

## License

Apache-2.0
