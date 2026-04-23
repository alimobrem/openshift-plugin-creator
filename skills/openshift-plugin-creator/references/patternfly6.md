# PatternFly 6 Component Patterns Reference

## Unified Theme & Design Tokens

PatternFly 6 uses semantic design tokens instead of hardcoded values. This enables automatic dark mode support and consistent theming.

### Token Hierarchy (use in this order)

1. **Semantic tokens** (recommended): Named by function, theme-aware
2. **Base tokens** (use sparingly): Support semantic tokens
3. **Palette tokens** (avoid): Foundation only

### Key Tokens

```css
/* Background */
--pf-t--global--background--color              /* primary background */
--pf-t--global--background--color--secondary    /* secondary background */

/* Text */
--pf-t--global--text--color                     /* primary text */
--pf-t--global--text--color--subtle             /* secondary text */

/* Status */
--pf-t--global--color--status--success          /* green — healthy/ready */
--pf-t--global--color--status--warning          /* yellow — degraded/progressing */
--pf-t--global--color--status--danger           /* red — error/failed */
--pf-t--global--color--status--info             /* blue — informational */

/* Spacing */
--pf-t--global--spacer--xs                      /* 0.25rem */
--pf-t--global--spacer--sm                      /* 0.5rem */
--pf-t--global--spacer--md                      /* 1rem */
--pf-t--global--spacer--lg                      /* 1.5rem */
--pf-t--global--spacer--xl                      /* 2rem */

/* Borders */
--pf-t--global--border--color                   /* default border */
--pf-t--global--border--width--default          /* default border width */
```

### Rules

- Never use hex colors — always use semantic tokens
- Never use hardcoded pixel values for spacing — use spacer tokens
- Dark mode is automatic when tokens are used correctly
- Prefix all custom CSS classes with the plugin name (e.g., `console-plugin-nhc__list-page`)
- Never use `.pf-` or `.co-` class prefixes (reserved)
- No naked element selectors in CSS — only class selectors

---

## Page Layout

```typescript
import {
  Page,
  PageSection,
  Content,
} from '@patternfly/react-core';

const MyPage: React.FC = () => (
  <>
    <PageSection variant="light">
      <Content>
        <h1>{t('Page Title')}</h1>
      </Content>
    </PageSection>
    <PageSection>
      {/* Main content */}
    </PageSection>
  </>
);
```

## Table (List View Pattern)

```typescript
import {
  Toolbar,
  ToolbarContent,
  ToolbarItem,
  SearchInput,
  Label,
  Button,
} from '@patternfly/react-core';
import {
  Table,
  Thead,
  Tbody,
  Tr,
  Th,
  Td,
  ThProps,
} from '@patternfly/react-table';

// Sortable table with toolbar
const ListPage: React.FC = () => {
  const { t } = useTranslation('plugin__my-plugin');
  const [activeNamespace] = useActiveNamespace();
  const [resources, loaded, error] = useK8sWatchResource<MyResource[]>({
    groupVersionKind: myResourceGVK,
    isList: true,
    namespace: activeNamespace === '#ALL_NS#' ? undefined : activeNamespace,
  });

  const [sortIndex, setSortIndex] = React.useState(0);
  const [sortDirection, setSortDirection] = React.useState<'asc' | 'desc'>('asc');
  const [filter, setFilter] = React.useState('');

  const getSortParams = (columnIndex: number): ThProps['sort'] => ({
    sortBy: { index: sortIndex, direction: sortDirection },
    onSort: (_event, index, direction) => {
      setSortIndex(index);
      setSortDirection(direction);
    },
    columnIndex,
  });

  const filtered = resources
    ?.filter((r) => r.metadata.name.includes(filter))
    .sort(/* sort logic based on sortIndex/sortDirection */);

  return (
    <>
      <PageSection variant="light">
        <Content><h1>{t('My Resources')}</h1></Content>
      </PageSection>
      <PageSection>
        <Toolbar>
          <ToolbarContent>
            <ToolbarItem>
              <SearchInput
                placeholder={t('Filter by name...')}
                value={filter}
                onChange={(_event, value) => setFilter(value)}
                onClear={() => setFilter('')}
              />
            </ToolbarItem>
            <ToolbarItem>
              <Button variant="primary">{t('Create')}</Button>
            </ToolbarItem>
          </ToolbarContent>
        </Toolbar>
        <Table>
          <Thead>
            <Tr>
              <Th sort={getSortParams(0)}>{t('Name')}</Th>
              <Th>{t('Namespace')}</Th>
              <Th sort={getSortParams(2)}>{t('Status')}</Th>
              <Th>{t('Created')}</Th>
              <Td />
            </Tr>
          </Thead>
          <Tbody>
            {filtered?.map((resource) => (
              <Tr key={resource.metadata.uid}>
                <Td dataLabel={t('Name')}>
                  <ResourceLink
                    groupVersionKind={myResourceGVK}
                    name={resource.metadata.name}
                    namespace={resource.metadata.namespace}
                  />
                </Td>
                <Td dataLabel={t('Namespace')}>{resource.metadata.namespace}</Td>
                <Td dataLabel={t('Status')}>
                  <StatusLabel conditions={resource.status?.conditions} />
                </Td>
                <Td dataLabel={t('Created')}>
                  <Timestamp timestamp={resource.metadata.creationTimestamp} />
                </Td>
                <Td isActionCell>{/* Row actions dropdown */}</Td>
              </Tr>
            ))}
          </Tbody>
        </Table>
      </PageSection>
    </>
  );
};
```

## Status Label Pattern

```typescript
import { Label } from '@patternfly/react-core';
import {
  CheckCircleIcon,
  ExclamationCircleIcon,
  ExclamationTriangleIcon,
  InProgressIcon,
} from '@patternfly/react-icons';

type StatusLabelProps = {
  conditions?: Condition[];
};

const StatusLabel: React.FC<StatusLabelProps> = ({ conditions }) => {
  const readyCondition = conditions?.find((c) => c.type === 'Ready' || c.type === 'Available');

  if (!readyCondition) {
    return <Label color="grey">{t('Unknown')}</Label>;
  }

  if (readyCondition.status === 'True') {
    return <Label color="green" icon={<CheckCircleIcon />}>{t('Ready')}</Label>;
  }

  if (readyCondition.reason === 'Progressing') {
    return <Label color="blue" icon={<InProgressIcon />}>{t('Progressing')}</Label>;
  }

  if (readyCondition.status === 'False') {
    return <Label color="red" icon={<ExclamationCircleIcon />}>{t('Error')}</Label>;
  }

  return <Label color="orange" icon={<ExclamationTriangleIcon />}>{t('Degraded')}</Label>;
};
```

## Detail Page Pattern

```typescript
import {
  PageSection,
  Content,
  Tabs,
  Tab,
  TabTitleText,
  DescriptionList,
  DescriptionListGroup,
  DescriptionListTerm,
  DescriptionListDescription,
  Label,
  Flex,
  FlexItem,
  Button,
} from '@patternfly/react-core';
import { ResourceYAMLEditor, Timestamp } from '@openshift-console/dynamic-plugin-sdk';

const DetailsPage: React.FC = () => {
  const { t } = useTranslation('plugin__my-plugin');
  const [activeTab, setActiveTab] = React.useState(0);

  return (
    <>
      <PageSection variant="light">
        <Flex justifyContent={{ default: 'justifyContentSpaceBetween' }}>
          <FlexItem>
            <Content>
              <h1>{resource.metadata.name}</h1>
            </Content>
          </FlexItem>
          <FlexItem>
            <Button variant="primary">{t('Edit')}</Button>
            <Button variant="danger">{t('Delete')}</Button>
          </FlexItem>
        </Flex>
      </PageSection>
      <PageSection type="tabs">
        <Tabs activeKey={activeTab} onSelect={(_e, key) => setActiveTab(key as number)}>
          <Tab eventKey={0} title={<TabTitleText>{t('Overview')}</TabTitleText>}>
            <PageSection>
              <DescriptionList>
                <DescriptionListGroup>
                  <DescriptionListTerm>{t('Name')}</DescriptionListTerm>
                  <DescriptionListDescription>{resource.metadata.name}</DescriptionListDescription>
                </DescriptionListGroup>
                <DescriptionListGroup>
                  <DescriptionListTerm>{t('Namespace')}</DescriptionListTerm>
                  <DescriptionListDescription>{resource.metadata.namespace}</DescriptionListDescription>
                </DescriptionListGroup>
                <DescriptionListGroup>
                  <DescriptionListTerm>{t('Created')}</DescriptionListTerm>
                  <DescriptionListDescription>
                    <Timestamp timestamp={resource.metadata.creationTimestamp} />
                  </DescriptionListDescription>
                </DescriptionListGroup>
                {/* Additional spec fields */}
              </DescriptionList>
            </PageSection>
          </Tab>
          <Tab eventKey={1} title={<TabTitleText>{t('YAML')}</TabTitleText>}>
            <PageSection>
              <ResourceYAMLEditor initialResource={resource} />
            </PageSection>
          </Tab>
        </Tabs>
      </PageSection>
    </>
  );
};
```

## Form Pattern (Create/Edit)

```typescript
import {
  Form,
  FormGroup,
  TextInput,
  NumberInput,
  Switch,
  Select,
  SelectOption,
  MenuToggle,
  FormFieldGroup,
  FormFieldGroupHeader,
  ActionGroup,
  Button,
  Alert,
  PageSection,
  Content,
} from '@patternfly/react-core';

const ResourceForm: React.FC = () => {
  const { t } = useTranslation('plugin__my-plugin');
  const [name, setName] = React.useState('');
  const [error, setError] = React.useState<string | null>(null);
  const [isSubmitting, setIsSubmitting] = React.useState(false);

  const handleSubmit = async () => {
    setIsSubmitting(true);
    setError(null);
    try {
      const resource = {
        apiVersion: 'example.com/v1',
        kind: 'MyResource',
        metadata: { name, namespace: activeNamespace },
        spec: { /* form values */ },
      };
      await k8sCreate({ model: MyResourceModel, data: resource });
      navigate(/* detail page path */);
    } catch (e) {
      setError(e.message);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <>
      <PageSection variant="light">
        <Content><h1>{t('Create MyResource')}</h1></Content>
      </PageSection>
      <PageSection>
        {error && <Alert variant="danger" title={t('Error creating resource')}>{error}</Alert>}
        <Form>
          <FormGroup label={t('Name')} isRequired fieldId="name">
            <TextInput
              id="name"
              value={name}
              onChange={(_event, val) => setName(val)}
              isRequired
            />
          </FormGroup>

          {/* Enum field */}
          <FormGroup label={t('Type')} fieldId="type">
            <Select
              id="type"
              toggle={(toggleRef) => (
                <MenuToggle ref={toggleRef}>{selectedType || t('Select type')}</MenuToggle>
              )}
              onSelect={(_event, val) => setSelectedType(val as string)}
            >
              <SelectOption value="option1">{t('Option 1')}</SelectOption>
              <SelectOption value="option2">{t('Option 2')}</SelectOption>
            </Select>
          </FormGroup>

          {/* Boolean field */}
          <FormGroup label={t('Enabled')} fieldId="enabled">
            <Switch
              id="enabled"
              isChecked={enabled}
              onChange={(_event, val) => setEnabled(val)}
            />
          </FormGroup>

          {/* Nested object */}
          <FormFieldGroup
            header={<FormFieldGroupHeader titleText={{ text: t('Advanced Settings') }} />}
          >
            <FormGroup label={t('Replicas')} fieldId="replicas">
              <NumberInput
                id="replicas"
                value={replicas}
                onMinus={() => setReplicas(Math.max(0, replicas - 1))}
                onPlus={() => setReplicas(replicas + 1)}
                onChange={(event) => setReplicas(Number(event.target.value))}
                min={0}
              />
            </FormGroup>
          </FormFieldGroup>

          <ActionGroup>
            <Button
              variant="primary"
              onClick={handleSubmit}
              isLoading={isSubmitting}
              isDisabled={isSubmitting || !name}
            >
              {t('Create')}
            </Button>
            <Button variant="link" onClick={() => navigate(-1)}>
              {t('Cancel')}
            </Button>
          </ActionGroup>
        </Form>
      </PageSection>
    </>
  );
};
```

## Delete Modal Pattern

```typescript
import {
  Modal,
  ModalVariant,
  Button,
  TextInput,
  FormGroup,
  Alert,
} from '@patternfly/react-core';

const DeleteModal: React.FC<{ resource: MyResource; onClose: () => void }> = ({
  resource,
  onClose,
}) => {
  const { t } = useTranslation('plugin__my-plugin');
  const [confirmName, setConfirmName] = React.useState('');
  const [error, setError] = React.useState<string | null>(null);
  const [isDeleting, setIsDeleting] = React.useState(false);

  const handleDelete = async () => {
    setIsDeleting(true);
    try {
      await k8sDelete({ model: MyResourceModel, resource });
      onClose();
      navigate(/* list page */);
    } catch (e) {
      setError(e.message);
    } finally {
      setIsDeleting(false);
    }
  };

  return (
    <Modal
      variant={ModalVariant.small}
      title={t('Delete {{name}}?', { name: resource.metadata.name })}
      isOpen
      onClose={onClose}
      actions={[
        <Button
          key="delete"
          variant="danger"
          onClick={handleDelete}
          isDisabled={confirmName !== resource.metadata.name || isDeleting}
          isLoading={isDeleting}
        >
          {t('Delete')}
        </Button>,
        <Button key="cancel" variant="link" onClick={onClose}>
          {t('Cancel')}
        </Button>,
      ]}
    >
      {error && <Alert variant="danger" title={t('Error')}>{error}</Alert>}
      <p>{t('This action cannot be undone. Type the resource name to confirm:')}</p>
      <FormGroup label={t('Resource name')} fieldId="confirm-name">
        <TextInput
          id="confirm-name"
          value={confirmName}
          onChange={(_event, val) => setConfirmName(val)}
        />
      </FormGroup>
    </Modal>
  );
};
```

## Empty State Pattern

```typescript
import {
  EmptyState,
  EmptyStateBody,
  EmptyStateActions,
  EmptyStateFooter,
  Button,
} from '@patternfly/react-core';
import { PlusCircleIcon } from '@patternfly/react-icons';

const NoResources: React.FC = () => {
  const { t } = useTranslation('plugin__my-plugin');
  return (
    <EmptyState
      headingLevel="h2"
      icon={PlusCircleIcon}
      titleText={t('No MyResources found')}
    >
      <EmptyStateBody>
        {t('Create a MyResource to get started.')}
      </EmptyStateBody>
      <EmptyStateFooter>
        <EmptyStateActions>
          <Button variant="primary">{t('Create MyResource')}</Button>
        </EmptyStateActions>
      </EmptyStateFooter>
    </EmptyState>
  );
};
```

## Dashboard Card Pattern

```typescript
import {
  Card,
  CardTitle,
  CardBody,
  Flex,
  FlexItem,
  Label,
} from '@patternfly/react-core';

const OperatorDashboardCard: React.FC = () => {
  const { t } = useTranslation('plugin__my-plugin');
  const [resources, loaded] = useK8sWatchResource<MyResource[]>({
    groupVersionKind: myResourceGVK,
    isList: true,
  });

  const ready = resources?.filter(isReady).length ?? 0;
  const total = resources?.length ?? 0;

  return (
    <Card>
      <CardTitle>{t('My Operator')}</CardTitle>
      <CardBody>
        <Flex gap={{ default: 'gapMd' }}>
          <FlexItem>
            <Label color="green">{t('{{count}} Ready', { count: ready })}</Label>
          </FlexItem>
          <FlexItem>
            <Label color="grey">{t('{{count}} Total', { count: total })}</Label>
          </FlexItem>
        </Flex>
      </CardBody>
    </Card>
  );
};
```

## Layout Components Quick Reference

| Component | Import | Use for |
|-----------|--------|---------|
| `PageSection` | `@patternfly/react-core` | Top-level page sections |
| `Content` | `@patternfly/react-core` | Text content with typographic styles |
| `Flex` / `FlexItem` | `@patternfly/react-core` | Flexible layouts |
| `Stack` / `StackItem` | `@patternfly/react-core` | Vertical stacking |
| `Grid` / `GridItem` | `@patternfly/react-core` | CSS grid layouts (span 1-12) |
| `Gallery` / `GalleryItem` | `@patternfly/react-core` | Responsive card grids |
| `Card` | `@patternfly/react-core` | Content containers |
| `Divider` | `@patternfly/react-core` | Visual separators |
| `Panel` | `@patternfly/react-core` | Bordered content areas |
| `Sidebar` | `@patternfly/react-core` | Side panel layouts |
