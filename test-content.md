---
title: "API Bundle"
sidebar_label: 'API Bundle'
id: api-bundle
description: 'Traefik Hub API Bundle overview.'
tags:
- API
- API Management
- CRD
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

API Bundles allow you to group one or more APIs together, enabling efficient management through Plans.

## API Bundle Behaviour

An API Bundle acts as a facade for one or more APIs. Wherever an API can be referenced, an API Bundle can be referenced as well. This feature enables:

- Grouping multiple APIs under a single entity.
- Applying API Plans to API Bundles i.e governing rate limits and quotas across all included APIs.

:::warning Note
An API Bundle cannot include another API Bundle.
:::

### Creating an API Bundle

You can define an API Bundle using the `APIBundle` custom resource (CR). Currently, There are two ways to include APIs in a bundle:

<Tabs>
<TabItem value="By Explicit API Names">

```yaml showLineNumbers {7-9}
apiVersion: hub.traefik.io/v1alpha1
kind: APIBundle
metadata:
  name: my-bundle
  namespace: your-namespace
spec:
  apis:
    - name: api-weather
    - name: api-forecast
```

</TabItem>

<TabItem value="By Label Selector">

```yaml showLineNumbers {7-9}
apiVersion: hub.traefik.io/v1alpha1
kind: APIBundle
metadata:
  name: my-bundle
  namespace: your-namespace
spec:
  apiSelector:
    matchLabels:
      category: weather
```

</TabItem>

</Tabs>

- Explicitly defining the API name directly lists the APIs (`api-weather` and `api-forecast`) to include in `my-bundle`.
- Using the label selector method allows you to dynamically bundle APIs without listing each one explicitly.

### Granting access to an API Bundle

Use the [Managed Subscription](./managed-subscription.md) resource to grant applications access to an API Bundle **on the Gateway**.

```yaml showLineNumbers {11,12}
apiVersion: hub.traefik.io/v1alpha1
kind: ManagedSubscription
metadata:
  name: access-to-my-bundle
  namespace: your-namespace
spec:
  applications:
    - appId: "your-application-id"
    # List additional applications as needed
    # - appId: "another-application-id"
  apiBundles:
    - name: my-bundle
  apiPlan:
    name: std-plan
```

### Making an API Bundle visible in Developer Portal

To control the visibility of the API Bundle in the Developer Portal for specific user groups, use the [`APICatalogItem`](./api-catalogitem.md) resource.

```yaml showLineNumbers {11,12}
apiVersion: hub.traefik.io/v1alpha1
kind: APICatalogItem
metadata:
  name: my-bundle-catalog-item
  namespace: your-namespace
spec:
  groups:
    - external
  # Or, to make it visible to all users:
  # everyone: true
  apiBundles:
    - name: my-bundle
  apiPlan:
    name: std-plan
```

### Title

The APIBundle object supports a custom `title` field. This field allows you to specify a human-readable name for the API Bundle as it appears on the portal.

```yaml showLineNumbers {4}
apiVersion: hub.traefik.io/v1alpha1
kind: APIBundle
metadata:
  name: my-bundle
  namespace: your-namespace
spec:
  title: "My Custom API Bundle"
  apis:
    - name: api-weather
    - name: api-forecast
```

In this example, the APIBundle named `my-bundle` is given the public title “My Custom API Bundle”, which will be used in the API portal.

## API Bundle Example Use Cases

### Use Case 1: Partner Access to Bundled APIs

**Scenario:** You have two APIs, `Weather` and `Forecast`, that you want to offer to your partners as a single bundle called `WeatherBundle`.
You wish to enforce a rate limit of 10 requests per second and a monthly quota of 1,000 requests.

<Tabs>
<TabItem value="API Bundle">

```yaml showLineNumbers title="This bundle, named weather-bundle, explicitly lists api-weather and api-forecast as the APIs it includes"
apiVersion: hub.traefik.io/v1alpha1
kind: APIBundle
metadata:
  name: weather-bundle
  namespace: your-namespace
spec:
  apis:
    - name: api-weather
    - name: api-forecast
```

</TabItem>

<TabItem value="API Plan">

```yaml showLineNumbers title="The std-plan enforces a rate limit of 10 requests per second and a monthly quota of 1,000 requests"
apiVersion: hub.traefik.io/v1alpha1
kind: APIPlan
metadata:
  name: std-plan
  namespace: your-namespace
spec:
  title: "Standard Plan"
  description: "**Standard Plan with rate limits and quotas**"
  rateLimit:
    limit: 1
    period: 1s
  quota:
    limit: 10000
    period: 750h # Approximately one month
```

</TabItem>

<TabItem value="APICatalogItem">

```yaml showLineNumbers title="An APICatalogItem resource to make the bundle visible to the external group"
apiVersion: hub.traefik.io/v1alpha1
kind: APICatalogItem
metadata:
  name: external-weather-bundle
  namespace: your-namespace
spec:
  groups:
    - external
  apiBundles:
    - name: weather-bundle
  apiPlan:
    name: std-plan
```

</TabItem>

<TabItem value="ManagedSubscription">

```yaml showLineNumbers title="Use ManagedSubscription to grant consumption access to partners' applications"
apiVersion: hub.traefik.io/v1alpha1
kind: ManagedSubscription
metadata:
  name: weather-bundle-subscription
  namespace: your-namespace
spec:
  applications:
    - appId: "partner-app-1"
    - appId: "partner-app-2"
  apiBundles:
    - name: weather-bundle
  apiPlan:
    name: std-plan
  weight: 100
```

</TabItem>
</Tabs>

#### Resulting Behavior

- Partners in the External Group: When users in the external group access the Developer Portal, they will see the `weather-bundle` available to them.
- Consumption Access: Applications with `appIds` `partner-app-1` and `partner-app-2` are granted access to the weather-bundle under the `std-plan`.
- Shared Rate Limits and Quotas: All APIs within weather-bundle share the same rate limits and quotas defined by std-plan because access was granted under the same [ManagedSubscription](./managed-subscription.md).
- Quota Renewal: The quota of 10,000 requests is enforced over a sliding window of approximately one month (750 hours). As older requests move out of the window, new requests become available,
allowing continuous usage up to the quota limit.

### Use Case 2: Internal Applications with Access to Individual APIs and Bundles

**Scenario**: Internal users & applications need access to all APIs, including an Admin API, with higher rate limits and quotas. The Admin API should be accessible both individually and as part of a bundle.

<Tabs>
<TabItem value="Weather API">

```yaml
apiVersion: hub.traefik.io/v1alpha1
kind: API
metadata:
  name: api-weather
  namespace: your-namespace
spec: {}  # Additional specifications for the Weather API
```

</TabItem>

<TabItem value="Forecast API">

```yaml
apiVersion: hub.traefik.io/v1alpha1
kind: API
metadata:
  name: api-forecast
  namespace: your-namespace
spec: {}   # Additional specifications for the forecast API
```

</TabItem>

<TabItem value="Admin API">

```yaml
apiVersion: hub.traefik.io/v1alpha1
kind: API
metadata:
  name: api-admin
  namespace: your-namespace
spec: {} # Additional specifications for the admin API
```

</TabItem>

<TabItem value="API Bundle">

```yaml showLineNumbers title="The website-apis bundle includes api-weather, api-forecast, and api-admins"
apiVersion: hub.traefik.io/v1alpha1
kind: APIBundle
metadata:
  name: website-apis
  namespace: your-namespace
spec:
  apis:
    - name: api-weather
    - name: api-forecast
    - name: api-admin
```

</TabItem>

<TabItem value="API Plan">

```yaml showLineNumbers title="The internal-plan provides a higher rate limit (50 requests per second) and the same monthly quota as the standard plan."
apiVersion: hub.traefik.io/v1alpha1
kind: APIPlan
metadata:
  name: internal-plan
  namespace: your-namespace
spec:
  title: "Internal Plan"
  description: "**Higher limits for internal developers**"
  rateLimit:
    limit: 50
    period: 1s
  quota:
    limit: 10000
    period: 750h # Approximately one month
```

</TabItem>

<TabItem value="APICatalogitem">

```yaml showLineNumbers title="APICatalogItem to make API visible to users in the 'Internal' group."
apiVersion: hub.traefik.io/v1alpha1
kind: APICatalogItem
metadata:
  name: internal-api-catalog
  namespace: your-namespace
spec:
  groups:
    - internal
  apis:
    - name: api-admin
  apiBundles:
    - name: website-apis
  apiPlan:
    name: internal-plan
```

</TabItem>

<TabItem value="ManagedSubscription">

```yaml showLineNumbers title="ManagedSubscription granting access to internal applications"
apiVersion: hub.traefik.io/v1alpha1
kind: ManagedSubscription
metadata:
  name: internal-api-subscription
  namespace: your-namespace
spec:
  applications:
    - appId: "internal-app-1"
    - appId: "internal-app-2"
  apis:
    - name: api-admin
  apiBundles:
    - name: website-apis
  apiPlan:
    name: internal-plan
  weight: 100
```

</TabItem>

</Tabs>

#### Resulting Behaviour

- Multiple Access Points: Internal developers can access the Admin API either directly via `api-admin` or through the `website-apis` bundle on the Developer Portal.
- Higher Limits: Applications (`internal-app-1`, `internal-app-2`) benefit from the higher rate limits and quotas specified in the internal-plan.
- Visibility and Access:
  - Visibility: Users in the internal group see all APIs (`api-weather`, `api-forecast`, `api-admin`) and the `website-apis` bundle on the Developer Portal, as defined in the `APICatalogItem`.
  - Access: The specified applications have consumption access to the APIs and bundle under the `internal-plan`, as defined in the `ManagedSubscription`.

### Use Case 3: Users in Multiple Groups With Different Access Levels

**Scenario:** Some users belong to both the `external` and `vip` groups. You want VIP users to have higher rate limits and quotas when accessing `WeatherBundle`.

<Tabs>
<TabItem value="API Plan">

```yaml showLineNumbers title="The extra-plan offers higher rate limits (20 requests per second) and quotas for VIP users"
apiVersion: hub.traefik.io/v1alpha1
kind: APIPlan
metadata:
  name: extra-plan
  namespace: your-namespace
spec:
  title: "VIP Plan"
  description: "**Enhanced limits for VIP users**"
  rateLimit:
    limit: 20
    period: 1s
  quota:
    limit: 20000
    period: 750h # Approximately one month
```

</TabItem>

<TabItem value="APICatalogItem for VIP Users">

```yaml showLineNumbers title="APICatalogItem to make weather-bundle visible in the Developer Portal to VIP group with extra plan"
apiVersion: hub.traefik.io/v1alpha1
kind: APICatalogItem
metadata:
  name: vip-weather-catalog
  namespace: your-namespace
spec:
  groups:
    - vip
  apiBundles:
    - name: weather-bundle
  apiPlan:
    name: extra-plan
```

</TabItem>

<TabItem value="APICatalogItem for External Users">

```yaml showLineNumbers title="APICatalogItem to make weather-bundle visible to in the Developer Portal to the external group with standard plan"
apiVersion: hub.traefik.io/v1alpha1
kind: APICatalogItem
metadata:
  name: external-weather-catalog
  namespace: your-namespace
spec:
  groups:
    - external
  apiBundles:
    - name: weather-bundle
  apiPlan:
    name: std-plan
```

</TabItem>

<TabItem value="ManagedSubscription for VIP Applications">

```yaml showLineNumbers title="ManagedSubscription granting higher limits to VIP Applications"
apiVersion: hub.traefik.io/v1alpha1
kind: ManagedSubscription
metadata:
  name: vip-weather-subscription
  namespace: your-namespace
spec:
  applications:
    - appId: "vip-app-1"
    - appId: "vip-app-2"
    # Add more VIP applications as needed
  apiBundles:
    - name: weather-bundle
  apiPlan:
    name: extra-plan
  weight: 2
```

</TabItem>

<TabItem value="ManagedSubscription for External Applications">

```yaml showLineNumbers title="ManagedSubscription for external applications with standard plan"
apiVersion: hub.traefik.io/v1alpha1
kind: ManagedSubscription
metadata:
  name: external-weather-subscription
  namespace: your-namespace
spec:
  applications:
    - appId: "external-app-1"
    - appId: "external-app-2"
    # Add more external applications as needed
  apiBundles:
    - name: weather-bundle
  apiPlan:
    name: std-plan
  weight: 1
```

</TabItem>

</Tabs>

#### Resulting Behavior

- Priority Determination: When an application matches multiple `ManagedSubscriptions` (e.g., an application belonging to both VIP and external users), the subscription with the higher weight takes precedence.
In this case, `vip-weather-subscription` has a weight of 2, which takes precedence over `external-weather-subscription` with a weight of 1.
- Higher Limits for VIP Applications: Applications listed in the VIP `ManagedSubscription` (`vip-app-1`, `vip-app-2`) benefit from the higher rate limits and quotas defined in the `extra-plan` when accessing the APIs within `weather-bundle`.
- Standard Limits for External Applications: Applications listed in the external `ManagedSubscription` (`external-app-1`, `external-app-2`) are subject to the standard limits defined in the `std-plan`.
- Visibility on Developer Portal:
  - Users in the `vip` group see the `weather-bundle` and `extra-plan` in the Developer Portal.
  - Users in the `external` group see weather-bundle with the std-plan available.
  - Users belonging to both groups see `weather-bundle` with both plans available.

## Related Content

- Learn more about the [API](./api.md) object in its dedicated section.
- Learn more about the [API Portal](./api-portal.md) in its dedicated section.
- Learn more about [User Management](./users-and-groups.md) in its dedicated section.