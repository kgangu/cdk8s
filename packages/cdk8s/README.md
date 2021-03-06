# cdk8s

> Cloud Development Kit for Kubernetes

**cdk8s** is a software development framework for defining Kubernetes
applications using rich object-oriented APIs. It allows developers to leverage
the full power of software in order to define abstract components called
"constructs" which compose Kubernetes resources or other constructs into
higher-level abstractions.

This library is the foundation of **cdk8s**. It includes base types that are
used to define cdk8s applications.

## Chart

The `Chart` is a container that synthesizes a single Kubernetes manifest.

```ts
class MyChart extends Chart {
  constructor(scope: Construct, ns: string) {
    super(scope, ns);

    // add contents here
  }
}
```

During synthesis, charts collect all the `ApiObject` nodes (recursively) and
emit a single YAML manifest that includes all these objects.

## ApiObject

An `ApiObject` is a construct that represents an entry in a Kubernetes manifest.
In most cases, you won't use `ApiObject` directly but rather use classes that
are generated by the cdk8s CLI and extend this base class.


## Include

The `Include` construct can be used to include an existing manifest in a chart.

The following example will include the Kubernetes Dashboard in `MyChart`:

```ts
import { Include } from 'cdk8s';

class MyChart extends Chart {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    const dashboard = new Include(this, 'dashboard', {
      url: 'https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml',
      // or
      url: 'dashboard.yaml'
    });

    // ...other resources
  }
}
```

All API objects defined in the included manifest will be added as children
`ApiObject`s under the `Include` construct's scope. This implies that you can
use `Node.of(include).children` to inspect them.

The following example queries for all the `Deployment` resources in the
dashboard:

```ts
const deps = Node.of(dashboard)
  .children
  .filter((c: ApiObject) => c.kind === 'Deployment');
```

NOTE: names of included objects (`metadata.name`) are preserved. This means that
if you try to include the same manifest twice into the same chart, your manifest
will have duplicate definitions of the same objects.

### Dependencies

You can declare dependencies between various `cdk8s` constructs by using the built-in support of the underlying `constructs` model.

#### ApiObjects

For example, you can force kubernetes to first apply a `Namespace` before applying the `Service` in the scope of that namespace:

```typescript

const namespace = new k8s.Namespace(chart, 'backend');
const service = new k8s.Service(chart, 'Service', { metadata: { namespace: namespace.name }});

// declare the dependency. this is just a syntactic sugar for Node.of(service).addDependency(namespace)
service.addDependency(namespace);
```

`cdk8s` will ensure that the `Namespace` object is placed before the `Service` object in the resulting manifest:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chart-backend-a59d2e47
---
apiVersion: v1
kind: Service
metadata:
  name: chart-service-93d02be7
  namespace: chart-backend-a59d2e47
```

#### Charts

You can also specify dependencies between charts, in exactly the same manner. For example, if we have a chart that provisions our `namespace`, we need that chart to be applied first:

```typescript
const namespaceChart = new NamespaceChart(app, 'namespace');
const applicationChart = new ApplicationChart(app, 'application');

// declare the dependency. this is just a syntactic sugar for Node.of(applicationChart).addDependency(namespaceChart)
applicationChart.addDependency(namespaceChart);
```

Running `cdk8s synth` will produce the following dist directory:

```console
> cdk8s synth

dist/0000-namespace.k8s.yaml
dist/0001-application.k8s.yaml
```

Notice that the `namespace` chart appears first with the `0000` prefix. This will ensure that a subsequent execution of `kubectl apply -f dist/` will apply the `namespace` first, and the `application` second.

#### Custom Constructs

The behavior above applies in the same way to custom constructs that you create or use.

```typescript
class Database extends Construct {
  constructor(scope: Construct, name: string) {
    super(scope, name);

    new k8s.StatefulSet(this, 'StatefulSet');
    new k8s.ConfigMap(this, 'ConfigMap');
  }
}

const app = new App();

const chart = new Chart(app, 'Chart');

const service = new k8s.Service(chart, 'Service')
const database = new Database(chart, 'Database');

service.addDependency(database);
```

Declaring such a dependency will cause **each** `ApiObject` in the source construct, to *depend on* **every** `ApiObject` in the target construct.

Note that in the example above, the source construct is actually an `ApiObject`, which is also ok since it is essentially a construct with a single `ApiObject`.

> Note that if the source of your dependency is a custom construct, it won't have the `addDependency` syntactic sugar by default, so you'll have to use `Node.of()`.

The resulting manifest will be:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: chart-database-statefulset-4627f8e2
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: chart-database-configmap-676f8640
---
apiVersion: v1
kind: Service
metadata:
  name: chart-service-93d02be7
```

You can see that all `ApiObject`s of the `Database` construct, appear before the `Service` object.

#### Things just got cool

If you simply declare a dependency between two `ApiObject`s (or `Constructs`), that belong to two different `Chart`s, `cdk8s` will create the chart dependency automatically for you.

```typescript
const namespaceChart = new NamespaceChart(app, 'namespace');
const applicationChart = new ApplicationChart(app, 'application');

const namespace = new k8s.Namespace(namespaceChart, 'namespace');
const deployment = new k8s.Deployment(applicationChart, 'Deployment');

// dependency between ApiObjects, not Charts!
deployment.addDependency(namespace);
```

Running `cdk8s synth` will produce the same result as if explicit chart dependencies were declared:

```console
> cdk8s synth

dist/0000-namespace.k8s.yaml
dist/0001-application.k8s.yaml
```

This means you need not be bothered with managing chart dependencies, simply work with the `ApiObject`s you create, and let `cdk8s` infer the chart dependencies.

### Testing

cdk8s bundles a set of test utilities under the `Testing` class:

* `Testing.app()` returns an `App` object bound to a temporary output directory.
* `Testing.synth(chart)` returns the Kubernetes manifest synthesized from a
  chart.


## License

This project is distributed under the [Apache License, Version 2.0](./LICENSE).

This module is part of the [cdk8s project](https://github.com/awslabs/cdk8s).
