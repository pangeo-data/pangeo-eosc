# Baremetal kubernetes deployment at CESNET for C-SCALE

These are the steps to deploy [Daskhub](https://docs.dask.org/en/stable/deploying-kubernetes-helm.html#helm-install-dask-for-multiple-users),
a [Dask Gateway](https://gateway.dask.org/) enabled [Jupyterhub](https://jupyter.org/hub) using the 
baremetal k8s cluster at [CESNET](https://www.cesnet.cz/?lang=en) in [C-SCALE](https://c-scale.eu/).

## Assumptions for redeployment

This guide assumes that you have:

1. Access to a kubernetes cluster with `kubectl`.
2. [Helm](https://helm.sh/) is installed (i.e. the `helm` command works).
3. Access to configure a DNS hostname for the **DaskHub** deployment. In this deployment we will use `pangeo.vm.fedcloud.eu`.

## Why not using the official helm chart?

The official helm chart uses [CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)
that are not allowed in the baremetal kubernetes deployment at CESNET. The modified chart
under [helm/daskhub/](helm/daskhub/) is adapted to avoid that issue.

## Before going ahead

Please update two fields in the `values.yaml` provided:

* `token1` and `token2` must be replaced with a hash generated using `openssl rand -hex 32` on Linux.
* `pangeo.vm.fedcloud.eu` must be replaced with your DNS hostname.

## Steps

Get a copy of this repository:

```bash
cd working/dir/
git clone git@github.com:pangeo-data/pangeo-eosc.git
```

Use `helm` to install **DaskHub** with the command below:

```bash
helm upgrade daskhub pangeo-eosc/helm/daskhub/ \
	--install --wait \
	--cleanup-on-fail \
	--create-namespace \
	--namespace c-scale-pangeo-dask \
	--version 2022.8.2 \
	--values pangeo-eosc/helm/daskhub/values.yaml
```

All going well **DaskHub** will be available at [https://pangeo.vm.fedcloud.eu](https://pangeo.vm.fedcloud.eu).
