# Console Plugin Template — File-by-File Reference

The template repo (`openshift/console-plugin-template`) provides the complete project skeleton. This reference documents every file and what to modify when scaffolding a new plugin.

## Root Configuration

### package.json

The central configuration file. Key sections to update:

```json
{
  "name": "console-plugin-<name>",
  "version": "0.0.1",
  "description": "OpenShift Console plugin for <Operator Name>",
  "license": "Apache-2.0",
  "consolePlugin": {
    "name": "console-plugin-<name>",
    "version": "0.0.1",
    "displayName": "<Human-Readable Plugin Name>",
    "exposedModules": {
      "MyResourceListPage": "./src/components/MyResource/MyResourceListPage",
      "MyResourceDetailsPage": "./src/components/MyResource/MyResourceDetailsPage",
      "MyResourceForm": "./src/components/MyResource/MyResourceForm"
    },
    "dependencies": {
      "@console/pluginAPI": "*"
    }
  }
}
```

**What to change:**
- `name` — plugin name
- `consolePlugin.name` — must match
- `consolePlugin.displayName` — shown in the console plugin list
- `consolePlugin.exposedModules` — one entry per component referenced in console-extensions.json

### console-extensions.json

Declares all extension points. Every `$codeRef` must reference a key in `exposedModules`.

```json
[
  {
    "type": "console.navigation/section",
    "properties": {
      "id": "my-operator-section",
      "perspective": "admin",
      "name": "%plugin__console-plugin-<name>~My Operator%"
    }
  },
  {
    "type": "console.navigation/href",
    "properties": {
      "id": "my-resource-nav",
      "perspective": "admin",
      "section": "my-operator-section",
      "name": "%plugin__console-plugin-<name>~My Resources%",
      "href": "/my-resources"
    }
  },
  {
    "type": "console.page/route",
    "properties": {
      "exact": true,
      "path": ["/ns/:ns/my-resources", "/all-namespaces/my-resources"],
      "component": { "$codeRef": "MyResourceListPage" }
    }
  },
  {
    "type": "console.page/route",
    "properties": {
      "exact": true,
      "path": "/ns/:ns/my-resources/:name",
      "component": { "$codeRef": "MyResourceDetailsPage" }
    }
  },
  {
    "type": "console.page/route",
    "properties": {
      "exact": true,
      "path": ["/ns/:ns/my-resources/~new", "/ns/:ns/my-resources/:name/edit"],
      "component": { "$codeRef": "MyResourceForm" }
    }
  }
]
```

### webpack.config.ts

Usually needs no changes. Key settings:
- Dev server runs on port 9001
- `ConsoleRemotePlugin()` handles module federation
- `CopyWebpackPlugin` copies `locales/` to dist
- Production: hashed filenames, no source maps
- Development: named chunks, source maps enabled

### tsconfig.json

Usually needs no changes. Key settings:
- Target: ES2021
- Module: ESNext
- JSX: react-jsx
- Strict: true
- Include: `src/**/*`

### i18next-parser.config.js

Update the `defaultNamespace`:
```javascript
module.exports = {
  defaultNamespace: 'plugin__console-plugin-<name>',
  // ...
};
```

## Source Code

### src/components/

This is where all plugin views go. Organize by resource:

```
src/
├── components/
│   ├── MyResourceA/
│   │   ├── MyResourceAListPage.tsx
│   │   ├── MyResourceADetailsPage.tsx
│   │   ├── MyResourceAForm.tsx
│   │   └── MyResourceADeleteModal.tsx
│   ├── MyResourceB/
│   │   └── ...
│   └── shared/
│       ├── StatusLabel.tsx
│       └── ConditionsTable.tsx
├── types/
│   ├── myResourceA.ts
│   └── myResourceB.ts
├── models/
│   └── index.ts          (K8sModel definitions)
└── utils/
    └── conditions.ts     (condition parsing helpers)
```

### Existing ExamplePage.tsx

The template includes `src/components/ExamplePage.tsx`. Remove it and replace with the actual plugin components. Also remove its `exposedModules` entry from `package.json` and its extensions from `console-extensions.json`.

## Styling

### CSS Files

Place CSS files next to their components:
```
src/components/MyResource/
├── MyResourceListPage.tsx
└── MyResourceListPage.css
```

Import in the component:
```typescript
import './MyResourceListPage.css';
```

### Stylelint Rules (enforced by .stylelintrc.yaml)

- No hex colors — use PatternFly CSS variables
- No naked element selectors — use class selectors only
- No `.pf-` or `.co-` prefixes — use `<plugin-name>__` prefix
- BEM-style naming encouraged: `console-plugin-nhc__list-page--header`

## Helm Chart

### charts/openshift-console-plugin/

**Chart.yaml** — Update:
```yaml
name: console-plugin-<name>
description: OpenShift Console plugin for <Operator Name>
```

**values.yaml** — Update:
```yaml
plugin:
  name: console-plugin-<name>
  image: quay.io/<org>/console-plugin-<name>:latest
  port: 9443
  replicas: 2
```

**templates/consoleplugin.yaml** — Usually no changes needed, references values.

**templates/configmap.yaml** — Nginx config, usually no changes needed.

**templates/deployment.yaml** — Deployment spec, usually no changes needed.

**templates/service.yaml** — Service config, usually no changes needed.

## Dockerfile

Multi-stage build, usually no changes needed:

```dockerfile
# Stage 1: Build
FROM registry.access.redhat.com/ubi9/nodejs-22:latest
RUN npm i -g corepack && corepack enable
COPY . /usr/src/app
WORKDIR /usr/src/app
RUN yarn install --immutable && yarn build

# Stage 2: Serve
FROM registry.access.redhat.com/ubi9/nginx-120:latest
COPY --from=0 /usr/src/app/dist /usr/share/nginx/html
USER 1001
CMD ["nginx", "-g", "daemon off;"]
```

## Internationalization

### Locale Files

Location: `locales/en/plugin__console-plugin-<name>.json`

Generated by running `yarn i18n`. Contains all translatable strings extracted from source.

### Using Translations

```typescript
import { useTranslation } from 'react-i18next';

const MyComponent: React.FC = () => {
  const { t } = useTranslation('plugin__console-plugin-<name>');
  return <h1>{t('My Title')}</h1>;
};
```

### In console-extensions.json

Use the format: `%plugin__console-plugin-<name>~Display Text%`

## Testing

### Integration Tests

Location: `integration-tests/`

Structure:
```
integration-tests/
├── cypress.config.ts
├── cypress/
│   ├── e2e/
│   │   └── my-resource.cy.ts
│   ├── fixtures/
│   │   └── sample-resource.yaml
│   └── support/
│       ├── commands.ts
│       └── e2e.ts
```

Run with: `yarn test-cypress-headless`

## Scripts

| Script | Purpose |
|--------|---------|
| `yarn start` | Dev server on port 9001 |
| `yarn build` | Production build to `dist/` |
| `yarn start-console` | Console bridge + plugin |
| `yarn lint` | ESLint + Stylelint + Prettier |
| `yarn i18n` | Extract translation strings |
| `yarn test-cypress-headless` | Headless Cypress E2E |
