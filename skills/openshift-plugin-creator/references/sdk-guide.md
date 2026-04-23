# OpenShift Console Dynamic Plugin SDK Reference

## Packages

| Package | Purpose |
|---------|---------|
| `@openshift-console/dynamic-plugin-sdk` | Core APIs, hooks, types |
| `@openshift-console/dynamic-plugin-sdk-webpack` | Webpack `ConsoleRemotePlugin` for builds |

## Extension Types

Extensions are declared in `console-extensions.json` and reference exposed modules via `$codeRef`.

### Navigation & Routing

```json
// Route — registers a page at a URL path
{
  "type": "console.page/route",
  "properties": {
    "exact": true,
    "path": "/example",
    "component": { "$codeRef": "ExamplePage" }
  }
}

// Namespaced route — automatically includes namespace prefix
{
  "type": "console.page/route",
  "properties": {
    "exact": true,
    "path": ["/ns/:ns/my-resource", "/all-namespaces/my-resource"],
    "component": { "$codeRef": "MyResourceListPage" }
  }
}

// Navigation link
{
  "type": "console.navigation/href",
  "properties": {
    "id": "my-resource",
    "perspective": "admin",
    "section": "my-section",
    "name": "%plugin__my-plugin~My Resource%",
    "href": "/my-resource"
  }
}

// Navigation section (group of links)
{
  "type": "console.navigation/section",
  "properties": {
    "id": "my-section",
    "perspective": "admin",
    "name": "%plugin__my-plugin~My Operator%"
  }
}
```

### Perspectives

```json
// Custom perspective
{
  "type": "console.perspective",
  "properties": {
    "id": "my-perspective",
    "name": "%plugin__my-plugin~My Perspective%",
    "icon": { "$codeRef": "PerspectiveIcon" },
    "landingPageURL": { "$codeRef": "PerspectiveLanding.getLandingURL" },
    "importRedirectURL": { "$codeRef": "PerspectiveLanding.getImportRedirectURL" }
  }
}
```

### Dashboard Extensions

```json
// Dashboard card
{
  "type": "console.dashboards/card",
  "properties": {
    "tab": "main",
    "position": "RIGHT",
    "component": { "$codeRef": "DashboardCard" }
  }
}
```

### Action Extensions

```json
// Action provider for a resource type
{
  "type": "console.action/resource-provider",
  "properties": {
    "model": { "group": "example.com", "version": "v1", "kind": "MyResource" },
    "provider": { "$codeRef": "MyResourceActions.useMyResourceActions" }
  }
}
```

## Key Extension Types Reference

| Type | Purpose |
|------|---------|
| `console.page/route` | Register a page at a URL path |
| `console.navigation/href` | Add a link to the navigation menu |
| `console.navigation/section` | Group navigation links under a section |
| `console.navigation/resource-ns` | Namespace-scoped resource link |
| `console.navigation/resource-cluster` | Cluster-scoped resource link |
| `console.perspective` | Create a custom console perspective |
| `console.dashboards/card` | Add a card to the dashboard |
| `console.action/provider` | Provide actions for any context |
| `console.action/resource-provider` | Provide actions for a specific resource type |
| `console.model-metadata` | Define metadata for a resource model |
| `console.flag` | Register a feature flag |
| `console.yaml-template` | Provide default YAML for creating resources |
| `console.resource/create` | Customize the create resource experience |
| `console.user-preferences/item` | Add user preference items |
| `console.telemetry/listener` | Listen to telemetry events |

## SDK Hooks & Functions

### Watching Resources

```typescript
import { useK8sWatchResource } from '@openshift-console/dynamic-plugin-sdk';

// Watch a list of resources
const [resources, loaded, error] = useK8sWatchResource<MyResource[]>({
  groupVersionKind: {
    group: 'example.com',
    version: 'v1',
    kind: 'MyResource',
  },
  isList: true,
  namespace: activeNamespace, // omit for cluster-scoped
});

// Watch a single resource
const [resource, loaded, error] = useK8sWatchResource<MyResource>({
  groupVersionKind: {
    group: 'example.com',
    version: 'v1',
    kind: 'MyResource',
  },
  name: 'my-resource-name',
  namespace: 'my-namespace',
});
```

### CRUD Operations

```typescript
import {
  k8sCreate,
  k8sUpdate,
  k8sPatch,
  k8sDelete,
  k8sGet,
  k8sList,
} from '@openshift-console/dynamic-plugin-sdk';

// Create
const result = await k8sCreate({
  model: MyResourceModel,
  data: resourceObject,
});

// Update (full replace)
const result = await k8sUpdate({
  model: MyResourceModel,
  data: updatedResourceObject,
});

// Patch (partial update)
const result = await k8sPatch({
  model: MyResourceModel,
  resource: existingResource,
  data: [
    { op: 'replace', path: '/spec/fieldName', value: 'new-value' },
  ],
});

// Delete
await k8sDelete({
  model: MyResourceModel,
  resource: resourceToDelete,
});

// Get single resource
const resource = await k8sGet({
  model: MyResourceModel,
  name: 'resource-name',
  ns: 'namespace',
});

// List resources
const resources = await k8sList({
  model: MyResourceModel,
  queryParams: { ns: 'namespace' },
});
```

### Namespace & Active Context

```typescript
import { useActiveNamespace } from '@openshift-console/dynamic-plugin-sdk';

const [activeNamespace, setActiveNamespace] = useActiveNamespace();
// activeNamespace is the currently selected namespace
// '#ALL_NS#' means all namespaces
```

### YAML Editor

```typescript
import { ResourceYAMLEditor } from '@openshift-console/dynamic-plugin-sdk';

<ResourceYAMLEditor
  initialResource={resource}
  onSave={(content: string) => handleSave(content)}
/>
```

### Other Useful Hooks

```typescript
import {
  useK8sModel,           // Get a K8s model by GVK
  useK8sModels,          // Get all K8s models
  useActiveCluster,      // Get active cluster info
  useListPageFilter,     // Built-in list page filtering
  ListPageHeader,        // Standard list page header
  ListPageBody,          // Standard list page body
  ListPageCreate,        // Create button for list pages
  ListPageFilter,        // Filter toolbar for list pages
  ResourceLink,          // Link to a resource with icon
  ResourceIcon,          // Icon for a resource kind
  Timestamp,             // Formatted timestamp display
  StatusPopupSection,    // Status popup content
  StatusPopupItem,       // Individual status item
  NamespaceBar,          // Namespace selector bar
  HorizontalNav,         // Horizontal tab navigation
  DocumentTitle,         // Set the browser tab title
} from '@openshift-console/dynamic-plugin-sdk';
```

## Model Definition Pattern

```typescript
import { K8sModel } from '@openshift-console/dynamic-plugin-sdk';

export const MyResourceModel: K8sModel = {
  apiVersion: 'v1',
  apiGroup: 'example.com',
  kind: 'MyResource',
  plural: 'myresources',
  label: 'MyResource',
  labelPlural: 'MyResources',
  abbr: 'MR',
  namespaced: true,
};

export const myResourceGVK = {
  group: 'example.com',
  version: 'v1',
  kind: 'MyResource',
};
```

## Shared Modules

These are provided by the console at runtime — do NOT bundle them:

- `react`, `react-dom`, `react-router`, `react-router-dom`
- `react-redux`, `redux`, `redux-thunk`
- `react-i18next`, `i18next`
- `@patternfly/react-core`, `@patternfly/react-table`, `@patternfly/react-icons`
- `@patternfly/react-topology`
- `@openshift-console/dynamic-plugin-sdk`

The `ConsoleRemotePlugin` webpack plugin handles this automatically via module federation.

## Console Extensions JSON Rules

- Every `$codeRef` must reference a key in `package.json`'s `consolePlugin.exposedModules`
- Format: `{ "$codeRef": "ModuleName" }` for default export or `{ "$codeRef": "ModuleName.namedExport" }` for named exports
- i18n strings use format: `%plugin__<plugin-name>~Display Text%`
- Changes to `console-extensions.json` require restarting the dev server (no hot reload)
