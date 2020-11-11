# KubeCon 2020 Demo Operator

This operator project was built with [`operator-sdk`](https://sdk.operatorframework.io) for Operator SDK's OLM integration demo for KubeCon 2020.

### Prerequisites

Install [`kind`](https://github.com/kubernetes-sigs/kind/releases), [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/),
and [`operator-sdk`](https://github.com/operator-framework/operator-sdk/releases).

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

Coming soon.
