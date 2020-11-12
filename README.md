# KubeCon 2020 Demo Operator

This operator project was built with [`operator-sdk`](https://sdk.operatorframework.io)
for Operator SDK's OLM integration demo for KubeCon 2020.

### Prerequisites

Install [`kind`](https://github.com/kubernetes-sigs/kind/releases), [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/),
and [`operator-sdk`](https://github.com/operator-framework/operator-sdk/releases) for your host platform.

### Test

Create the test cluster, then install OLM and Prometheus:

```sh
kind create cluster --image=kindest/node:v1.18.8
kubectl apply -f https://raw.githubusercontent.com/coreos/prometheus-operator/release-0.33/bundle.yaml
operator-sdk olm install
```

Next, run the v0.0.1 bundle image created from this repository (present remotely):

```sh
kind load docker-image quay.io/estroz/memcached-operator:v0.0.1
operator-sdk run bundle quay.io/estroz/memcached-operator-bundle:v0.0.1
```

Now try creating some sample Memcached CR's:

```console
$ kubectl apply -f config/samples/invalid_even_size.yaml
Error from server (Cluster size must be an odd number): error when creating "config/samples/invalid_even_size.yaml": admission webhook "vmemcached.kb.io" denied the request: Cluster size must be an odd number
$ kubectl apply -f config/samples/valid_zero_size.yaml
memcacheds.cache.my.domain/memcached-sample-zero-size created
```

#### Cleanup

```sh
operator-sdk cleanup memcached-operator
operator-sdk olm uninstall
```

### Prepare for prod

#### More prerequisites

Install [`opm`](https://github.com/operator-framework/operator-registry/releases/) and
the [`kubectl` operator plugin](https://github.com/operator-framework/kubectl-operator/releases)
via [krew](https://krew.sigs.k8s.io/docs/user-guide/setup/install/) for your platform.

---

Create the test cluster, then install OLM and Prometheus:

```sh
kind create cluster --image=kindest/node:v1.18.8
kubectl apply -f https://raw.githubusercontent.com/coreos/prometheus-operator/release-0.33/bundle.yaml
operator-sdk olm install
```

Using the v0.0.2 memcached-operator bundle image, create an index image (catalog):

```sh
opm index add \
  --bundles quay.io/estroz/memcached-operator-bundle:v0.0.1,quay.io/estroz/memcached-operator-bundle:v0.0.2 \
  --tag quay.io/estroz/memcached-operator-index:latest
```

This index already exists remotely. If it didn't you could push it like any other image:

```sh
make docker-push IMG=quay.io/estroz/memcached-operator-index:latest
```

Now we add the memcached-operator catalog by creating an in-cluster resource:

```sh
kubectl operator catalog add memcached-operator quay.io/estroz/memcached-operator-index:latest
```

<!--

TODO: `kubectl operator upgrade` appears to be broken for webhook-enabled operators

##### Install an arbitrary version then upgrade

We want to simulate a real cluster environment in this scenario,
so we install an older memcached-operator version and apply a Memcached CR:

```sh
kubectl operator install memcached-operator \
  --version v0.0.1 \
  --channel alpha \
  --create-operator-group
```

We confirm v0.0.1 was installed by checking the operator's deployment image,
and create a Memcached CR:

```console
$ kubectl get deployment memcached-operator-controller-manager --template '{{ range .spec.template.spec.containers }}{{ if eq .name "manager" }}{{ .image }}{{"\n"}}{{ end }}{{ end }}'
quay.io/estroz/memcached-operator:v0.0.1
$ kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
memcacheds.cache.my.domain/memcached-sample created
```

Now we upgrade to the latest version, v0.0.2:

```sh
kubectl operator upgrade memcached-operator --channel alpha
```

We confirm v0.0.2 was upgraded to by checking the operator's deployment image again,
and make sure the Memcached CR is still available:

```console
$ kubectl get deployment memcached-operator-controller-manager --template '{{ range .spec.template.spec.containers }}{{ if eq .name "manager" }}{{ .image }}{{"\n"}}{{ end }}{{ end }}'
quay.io/estroz/memcached-operator:v0.0.2
$ kubectl get memcacheds memcached-sample --template='{{ .spec.size }}{{ "\n" }}'
3
```

##### Install channel head

-->

In this scenario, we want to freshly install an operator to a channel head.
This is typically how first-time operator installation is done.

Deploy the memcached-operator catalog with `kubectl operator`,
with automatic upgrades to the `alpha` channel head, the default channel:

```sh
kubectl operator install memcached-operator \
  --approval Automatic \
  --channel alpha \
  --create-operator-group
```

The version of the channel head, v0.0.2, should be installed. To ensure this is the case,
check the deployment's manager image version and try creating a Memcached CR:

```sh
$ kubectl get deployment memcached-operator-controller-manager --template '{{ range .spec.template.spec.containers }}{{ if eq .name "manager" }}{{ .image }}{{"\n"}}{{ end }}{{ end }}'
quay.io/estroz/memcached-operator:v0.0.2
$ kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
memcacheds.cache.my.domain/memcached-sample created
```

#### Cleanup

```sh
kubectl operator uninstall memcached-operator --delete-all
kubectl operator catalog remove memcached-operator
operator-sdk cleanup memcached-operator
operator-sdk olm uninstall
```
