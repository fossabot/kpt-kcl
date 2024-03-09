# kpt-kcl-sdk

[![Go Report Card](https://goreportcard.com/badge/github.com/kcl-lang/kpt-kcl)](https://goreportcard.com/report/github.com/kcl-lang/kpt-kcl)
[![GoDoc](https://godoc.org/github.com/kcl-lang/kpt-kcl?status.svg)](https://godoc.org/github.com/kcl-lang/kpt-kcl)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://github.com/kcl-lang/kpt-kcl/blob/main/LICENSE)

[KCL](https://github.com/kcl-lang/kcl) is a constraint-based record & functional domain language. Full documents of KCL can be found [here](https://kcl-lang.io/).

[kpt](https://github.com/GoogleContainerTools/kpt) is a package-centric toolchain that enables a WYSIWYG configuration authoring, automation, and delivery experience, which simplifies managing Kubernetes platforms and KRM-driven infrastructure.

The KPT KCL function SDK contains a KCL interpreter to run a KCL script to mutate or validate resources.

The KCL programming language can be used to:

+ Add annotations on the basis of a condition.
+ Inject a sidecar container in all KRM resources that contain a PodTemplate.
+ Validate all KRM resources using KCL schema.

## Prerequisites

+ Install [kpt](https://github.com/GoogleContainerTools/kpt)
+ Install [Docker](https://www.docker.com/)
+ Golang (at least version 1.21)

## Test the KRM function

You need to put your KCL script source in the functionConfig of kind KCLRun and then the function will run the KCL script that you provide.

This function can be used both declaratively and imperatively.

```bash
kpt fn source ./testdata/resources.yaml --fn-config ./testdata/fn-config.yaml | go run main.go
```

The output is:

```yaml
apiVersion: config.kubernetes.io/v1
kind: ResourceList
items:
- apiVersion: apps/v1
  kind: Deployment
  metadata:
    annotations:
      config.kubernetes.io/index: '1'
      config.kubernetes.io/path: resources.yaml
      internal.config.kubernetes.io/index: '1'
      internal.config.kubernetes.io/path: resources.yaml
      internal.config.kubernetes.io/seqindent: compact
    labels:
      app: nginx
    name: nginx-deployment
  spec:
    replicas: '5'
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - image: nginx:1.14.2
          name: nginx
          ports:
          - containerPort: 80
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      config.kubernetes.io/index: '0'
      config.kubernetes.io/path: resources.yaml
      internal.config.kubernetes.io/index: '0'
      internal.config.kubernetes.io/path: resources.yaml
      internal.config.kubernetes.io/seqindent: wide
    name: test
  spec:
    ports:
    - port: 80
      protocol: TCP
      targetPort: 9376
    selector:
      app: MyApp
functionConfig:
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: set-replicas
    annotations:
      config.kubernetes.io/index: '0'
      config.kubernetes.io/path: 'fn-config.yaml'
      internal.config.kubernetes.io/index: '0'
      internal.config.kubernetes.io/path: 'fn-config.yaml'
      internal.config.kubernetes.io/seqindent: 'compact'
  data:
    replicas: "5"
    source: |
      resources = option("resource_list")
      setReplicas = lambda items, replicas {
         [item | {if item.kind == "Deployment": spec.replicas = replicas} for item in items]
      }
      setReplicas(resources.items or [], resources.functionConfig.data.replicas)
```

Thus, the `spec.replicas` of `Deployment` in the `resource_list.yaml` is changed to `5` from `2`.

## FunctionConfig

There are 2 kinds of `functionConfig` supported by this function:

+ ConfigMap
+ A custom resource of kind `KCLRun`

To use a ConfigMap as the functionConfig, the KCL script source must be specified in the data.source field. Additional parameters can be specified in the data field.

Here's an example:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: set-replicas
data:
  replicas: "5"
  source: |
    resources = option("resource_list")
    setReplicas = lambda items, replicas {
       [item | {if item.kind == "Deployment": spec.replicas = replicas} for item in items]
    }
    setReplicas(resources.items or [], resources.functionConfig.data.replicas)
```

In the example above, the script accesses the replicas parameters using `option("resource_list").functionConfig.data.replicas`.

To use a KCLRun as the functionConfig, the KCL source must be specified in the source field. Additional parameters can be specified in the params field. The params field supports any complex data structure as long as it can be represented in YAML.

```yaml
apiVersion: krm.kcl.dev/v1alpha1
kind: KCLRun
metadata:
  name: conditionally-add-annotations
spec:
  params:
    toMatch:
      config.kubernetes.io/local-config: "true"
    toAdd:
      configmanagement.gke.io/managed: disabled
  source: |
    resource = option("resource_list")
    items = resource.items
    params = resource.functionConfig.spec.params
    toMatch = params.toMatch
    toAdd = params.toAdd
    [item | {
       # If all annotations are matched, patch more annotations
       if all key, value in toMatch {
          item.metadata.annotations[key] == value
       }:
           metadata.annotations: toAdd
    } for item in items]
```

In the example above, the script accesses the `toMatch` parameters using `option("resource_list").functionConfig.spec.params.toMatch`.

## Integrate the Function into kpt

### Build from Source

Firstly, build the container image from the source.

```bash
export FN_CONTAINER_REGISTRY=<Your GCR or docker hub>
export TAG=<Your KRM function tag>
docker build . -t ${FN_CONTAINER_REGISTRY}/${FUNCTION_NAME}:${TAG}
```

There are 2 ways to run the function declaratively.

+ Have your Kptfile with the inline ConfigMap as the functionConfig.
+ Have your Kptfile pointing to a functionConfig file that contains either a ConfigMap or a KCLRun.

After that, you can render it in the folder that contains KRM with:

```bash
kpt fn render
```

There are 2 ways to run the function imperatively.

+ Run it using a ConfigMap that is generated from the command line arguments. The KCL script lives in `main.k` file.
  
```bash
kpt fn eval --image ${FN_CONTAINER_REGISTRY}/${FUNCTION_NAME}:${TAG} -- source="$(cat main.k)" param1=value1 param2=value2
```

+ Or use the function config file.

```bash
kpt fn eval --image ${FN_CONTAINER_REGISTRY}/${FUNCTION_NAME}:${TAG} --fn-config fn-config.yaml
```

### Using the Pre-release KCL KPT Image

But for example, you can use the unstable kcl-kpt image `docker.io/kcllang/kpt-kcl:v0.3.0` for testing.

```bash
kpt fn eval ./testdata/resources.yaml -i docker.io/kcllang/kpt-kcl:v0.3.0 --fn-config ./testdata/fn-config.yaml
```

Then the Kubernetes resource file `resources.yaml` will be modified in place.

## Guides for Developing KCL

Here's what you can do in the KCL script:

+ Read resources from `option("resource_list")`. The `option("resource_list")` complies with the [KRM Functions Specification](https://github.com/kubernetes-sigs/kustomize/blob/master/cmd/config/docs/api-conventions/functions-spec.md#krm-functions-specification). You can read the input resources from `option("resource_list")["items"]` and the `functionConfig` from `option("resource_list")["functionConfig"]`.
+ Return a KPM list for output resources.
+ Return an error using `assert {condition}, {error_message}`.
+ Read the environment variables. e.g. `option("PATH")` (Not yet implemented).
+ Read the OpenAPI schema. e.g. `option("open_api")["definitions"]["io.k8s.api.apps.v1.Deployment"]` (Not yet implemented).

## Examples

See [here](https://kcl-lang.io/krm-kcl/tree/main/examples) for more examples.

## Library

You can directly use [KCL standard libraries](https://kcl-lang.io/docs/reference/model/overview) without importing them, such as `regex.match`, `math.log`.
