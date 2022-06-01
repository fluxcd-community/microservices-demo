# microservices-demo

Microservices demo made with
[podinfo](https://github.com/stefanprodan/podinfo),
managed by [flux](https://github.com/fluxcd/flux2)
and monitored by [weave-gitops](https://github.com/weaveworks/weave-gitops).

![](docs/img/weave-gitops-msdemo.png)

The microservices demo is composed of 20 Kubernetes Deployments
with a total request of `200m` CPU and `320Mi` memory.
Each microservice is managed by a dedicated Flux Kustomization, and it contains
a podinfo instance and a redis instance. The microservices are configured to
scale to a maximum of `2` pods, using CPU-based horizontal pod autoscalers.

![](docs/img/linkerd-msdemo.png)

The microservices demo comes with a client app that generates HTTP/S
traffic between microservices at a total rate of `30` requests/second.
Each microservice writes data in their dedicated Redis instance every `30s`

## Prerequisites 

* A Kubernetes cluster [bootstrapped with Flux](https://fluxcd.io/docs/installation/).
* Weave GitOps UI [HelmRelease deployed on the cluster](https://docs.gitops.weave.works/docs/getting-started/).

## Deploy microservices

Add the following definitions to the bootstrap repo under a cluster e.g. `clusters/my-cluster/msdemo.yaml`:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: msdemo
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    toolkit.fluxcd.io/tenant: msdemo
  name: flux
  namespace: msdemo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    toolkit.fluxcd.io/tenant: msdemo
  name: flux
  namespace: msdemo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: flux
    namespace: msdemo
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: msdemo
  namespace: msdemo
spec:
  interval: 1m0s
  ref:
    branch: main
  url: https://github.com/fluxcd-community/microservices-demo
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: msdemo
  namespace: msdemo
spec:
  targetNamespace: msdemo
  interval: 60m0s
  retryInterval: 1m30s
  path: ./deploy
  prune: true
  wait: true
  timeout: 2m
  serviceAccountName: flux
  sourceRef:
    kind: GitRepository
    name: msdemo
  postBuild:
    substitute:
      app_namespace: msdemo
  patches:
    - target:
        kind: Kustomization
      patch: |
        - op: add
          path: /spec/serviceAccountName
          value: flux
```

Note that the above configuration is compatible with Flux
[multi-tenancy lockdown mode](https://fluxcd.io/docs/installation/#multi-tenancy-lockdown).

To spin up multiple stacks, make a copy the above file, replace `msdemo` with `msdemo1` in
the multi-doc YAML and add it to your repository.

## Update microservices

To trigger a rolling deployment of all microservices, add the following patch to your `msdemo` Kustomization,
and set the [podinfo](https://github.com/stefanprodan/podinfo/releases) version to value greater than `6.1.3`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: msdemo
  namespace: msdemo
spec:
  patches:
    - target:
        kind: Kustomization
      patch: |
        - op: add
          path: /spec/postBuild/substitute/app_version
          value: 6.1.5
```

To update specific microservices, add their names to the patch target:

```yaml
  patches:
    - target:
        kind: Kustomization
        name: "(demo-frontend|demo-admin)"
      patch: |
        - op: add
          path: /spec/postBuild/substitute/app_version
          value: 6.1.6
```

To test rollout failures use a non existing version such as `99.0.0`.

## List microservices

The above configuration will deploy the following workloads:

```console
$ flux -n msdemo tree kustomization msdemo
Kustomization/msdemo/msdemo
├── Kustomization/msdemo/demo-admin
│   ├── ConfigMap/msdemo/demo-admin-redis
│   ├── Service/msdemo/demo-admin-app
│   ├── Service/msdemo/demo-admin-redis
│   ├── Deployment/msdemo/demo-admin-app
│   ├── Deployment/msdemo/demo-admin-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-admin-app
├── Kustomization/msdemo/demo-advert
│   ├── ConfigMap/msdemo/demo-advert-redis
│   ├── Service/msdemo/demo-advert-app
│   ├── Service/msdemo/demo-advert-redis
│   ├── Deployment/msdemo/demo-advert-app
│   ├── Deployment/msdemo/demo-advert-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-advert-app
├── Kustomization/msdemo/demo-auth
│   ├── ConfigMap/msdemo/demo-auth-redis
│   ├── Service/msdemo/demo-auth-app
│   ├── Service/msdemo/demo-auth-redis
│   ├── Deployment/msdemo/demo-auth-app
│   ├── Deployment/msdemo/demo-auth-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-auth-app
├── Kustomization/msdemo/demo-cart
│   ├── ConfigMap/msdemo/demo-cart-redis
│   ├── Service/msdemo/demo-cart-app
│   ├── Service/msdemo/demo-cart-redis
│   ├── Deployment/msdemo/demo-cart-app
│   ├── Deployment/msdemo/demo-cart-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-cart-app
├── Kustomization/msdemo/demo-catalogue
│   ├── ConfigMap/msdemo/demo-catalogue-redis
│   ├── Service/msdemo/demo-catalogue-app
│   ├── Service/msdemo/demo-catalogue-redis
│   ├── Deployment/msdemo/demo-catalogue-app
│   ├── Deployment/msdemo/demo-catalogue-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-catalogue-app
├── Kustomization/msdemo/demo-checkout
│   ├── ConfigMap/msdemo/demo-checkout-redis
│   ├── Service/msdemo/demo-checkout-app
│   ├── Service/msdemo/demo-checkout-redis
│   ├── Deployment/msdemo/demo-checkout-app
│   ├── Deployment/msdemo/demo-checkout-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-checkout-app
├── Kustomization/msdemo/demo-client
│   └── Deployment/msdemo/demo-client-app
├── Kustomization/msdemo/demo-frontend
│   ├── ConfigMap/msdemo/demo-frontend-redis
│   ├── Service/msdemo/demo-frontend-app
│   ├── Service/msdemo/demo-frontend-redis
│   ├── Deployment/msdemo/demo-frontend-app
│   ├── Deployment/msdemo/demo-frontend-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-frontend-app
├── Kustomization/msdemo/demo-mobile
│   ├── ConfigMap/msdemo/demo-mobile-redis
│   ├── Service/msdemo/demo-mobile-app
│   ├── Service/msdemo/demo-mobile-redis
│   ├── Deployment/msdemo/demo-mobile-app
│   ├── Deployment/msdemo/demo-mobile-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-mobile-app
├── Kustomization/msdemo/demo-payment
│   ├── ConfigMap/msdemo/demo-payment-redis
│   ├── Service/msdemo/demo-payment-app
│   ├── Service/msdemo/demo-payment-redis
│   ├── Deployment/msdemo/demo-payment-app
│   ├── Deployment/msdemo/demo-payment-redis
│   └── HorizontalPodAutoscaler/msdemo/demo-payment-app
└── Kustomization/msdemo/demo-shipping
    ├── ConfigMap/msdemo/demo-shipping-redis
    ├── Service/msdemo/demo-shipping-app
    ├── Service/msdemo/demo-shipping-redis
    ├── Deployment/msdemo/demo-shipping-app
    ├── Deployment/msdemo/demo-shipping-redis
    └── HorizontalPodAutoscaler/msdemo/demo-shipping-app
```
