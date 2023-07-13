# Baremetal kubernetes deployment at CESNET for C-SCALE

These are the steps to deploy [MinIO](https://min.io/),
[Jupyterhub](https://jupyter.org/hub),
[BinderHub](https://binderhub.readthedocs.io/),
and [Dask Gateway](https://gateway.dask.org/) using the
baremetal k8s cluster at [CESNET](https://www.cesnet.cz/?lang=en)
in [C-SCALE](https://c-scale.eu/).

## Assumptions for deployment

This guide assumes that you have:

1. Access to a kubernetes cluster with `kubectl`.
2. [Helm](https://helm.sh/) is installed (i.e. the `helm` command works).
3. Access to configure DNS hostnames (e.g.
   the [EGI Dynamic DNS](https://docs.egi.eu/users/compute/cloud-compute/dynamic-dns/))

## Get a copy of this repository

For example, using `git clone`, and make sure you are in the correct
workding directory:

```bash
cd working/dir/
git clone git@github.com:pangeo-data/pangeo-eosc.git
cd providers/kubernetes/
```

# DaskHub

To deploy [Jupyterhub](https://jupyter.org/hub) along with
[Dask Gateway](https://gateway.dask.org/) we use
[Daskhub](https://docs.dask.org/en/stable/deploying-kubernetes-helm.html#helm-install-dask-for-multiple-users).

## Why not using the official helm chart for DaskHub?

The official helm chart uses [CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)
that are not allowed in the baremetal kubernetes deployment at CESNET.
The modified chart under the [daskhub/](./daskhub/) folder is adapted
to avoid that issue.

## Before going ahead

Please configure the values below accordingly:

* [daskhub-secrets.yaml](./daskhub-secrets.yaml): `token1` and `token2` must be
  replaced with a hash generated using `openssl rand -hex 32` on Linux.
* [daskhub.yaml](./daskhub.yaml): `pangeo-eosc.vm.fedcloud.eu` must be replaced
  with your DNS hostname.

## Steps

Use `helm` to install **DaskHub** with the command below:

```bash
helm upgrade daskhub daskhub/ \
  --install --wait \
  --cleanup-on-fail \
  --create-namespace \
  --namespace c-scale-pangeo-dask \
  --version 2022.8.2 \
  --values daskhub.yaml \
  --values daskhub-secrets.yaml
```

All going well **DaskHub** will be available at [https://pangeo-eosc.vm.fedcloud.eu](https://pangeo-eosc.vm.fedcloud.eu).

# BinderHub and Dask Gateway

To get Dask Gateway working with Binder, first we deploy BinderHub and then Dask Gateway.

## Before going ahead

Please configure the values below accordingly:

* [binderhub-secrets.yaml](./binderhub/binderhub-secrets.yaml): `token1` and
  `token2` must be replaced with a hash generated using `openssl rand -hex 32`
  on Linux. Note that the same `token1` should also be configured in
  [dask-gateway-secrets.yaml](./binderhub/dask-gateway-secrets.yaml).
  You must also replace the credentials for the container registry
  (i.e. `registry.password` and `registry.username`), and finally
  `daskbinder.vm.fedcloud.eu` with the DNS hostname of your choice.
* [binderhub.yaml](./binderhub/binderhub.yaml): `daskbinder.vm.fedcloud.eu`
  and `daskgateway.vm.fedcloud.eu` must be replaced with your DNS hostname.

## Steps

Use `helm` to install **BinderHub** with the command below:

```bash
sudo helm repo add jupyterhub https://jupyterhub.github.io/helm-chart
sudo helm repo update
sudo helm upgrade binderhub jupyterhub/binderhub \
  --install \
  --cleanup-on-fail \
  --create-namespace \
  --namespace binderhub \
  --version 1.0.0-0.dev.git.3057.h0a4304c \
  --values binderhub/binderhub.yaml \
  --values binderhub/binderhub-secrets.yaml
```

Then deploy Dask Gateway following the steps in the
[documentation](https://gateway.dask.org/install-kube.html#install-the-dask-gateway-helm-chart):

```bash
sudo helm repo add dask https://helm.dask.org
sudo helm repo update
sudo helm upgrade dask-gateway dask-gateway \
  --repo=https://helm.dask.org \
  --install \
  --cleanup-on-fail \
  --create-namespace \
  --namespace binderhub \
  --version 2023.1.1 \
  --values binderhub/dask-gateway.yaml \
  --values binderhub/dask-gateway-secrets.yaml
```

All going well **BinderHub** will be available at [https://daskbinder.vm.fedcloud.eu](https://daskbinder.vm.fedcloud.eu).

# MinIO

## Before going ahead

Please configure the values below accordingly:

* [minio.yaml](./minio/minio.yaml): `pangeo-eosc-minio.vm.fedcloud.eu` and
  `pangeo-eosc-minioapi.vm.fedcloud.eu` must be replaced with your DNS hostname.
  Replace `pvc-minio-c-scale` with the corresponding
  [PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
  available on your baremetal k8s cluster.
* [minio-secrets.yaml](./minio/minio-secrets.yaml): `rootUser` and
  `rootPassword` must be replaced with your own. Please note that they need
  to be at least 3 and 8-character long, respectively.

## Steps

Use `helm` to install **MinIO** with the command below:

```bash
helm-v3.11.1/helm upgrade minio bitnami/minio \
  --install --wait \
  --cleanup-on-fail \
  --create-namespace \
  --namespace c-scale-pangeo-dask \
  --version 12.6.4 \
  --values minio/minio.yaml \
  --values minio/minio-secrets.yaml
```

All going well the **MinIO Console** will be available at
[https://pangeo-eosc-minio.vm.fedcloud.eu](https://pangeo-eosc-minio.vm.fedcloud.eu)
and the **MinIO API endpoint** will be available at
[https://pangeo-eosc-minioapi.vm.fedcloud.eu](https://pangeo-eosc-minioapi.vm.fedcloud.eu).
