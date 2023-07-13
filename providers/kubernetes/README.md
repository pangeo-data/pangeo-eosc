# Baremetal kubernetes deployment at CESNET for C-SCALE

These are the steps to deploy [MinIO](https://min.io/) and
[Daskhub](https://docs.dask.org/en/stable/deploying-kubernetes-helm.html#helm-install-dask-for-multiple-users),
a [Dask Gateway](https://gateway.dask.org/) enabled [Jupyterhub](https://jupyter.org/hub), using the
baremetal k8s cluster at [CESNET](https://www.cesnet.cz/?lang=en) in [C-SCALE](https://c-scale.eu/).

## Assumptions for redeployment

This guide assumes that you have:

1. Access to a kubernetes cluster with `kubectl`.
2. [Helm](https://helm.sh/) is installed (i.e. the `helm` command works).
3. Access to configure a DNS hostname for both **MinIO** and **DaskHub**.

## Why not using the official helm chart for DaskHub?

The official helm chart uses [CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions)
that are not allowed in the baremetal kubernetes deployment at CESNET.
The modified chart under the [daskhub/](./daskhub/) folder is adapted
to avoid that issue.

## Before going ahead

Please configure the values below accordingly:

* [daskhub.yaml](./daskhub.yaml): `token1` and `token2` must be replaced with
  a hash generated using `openssl rand -hex 32` on Linux.
* [daskhub-secrets.yaml](./daskhub-secrets.yaml): `pangeo-eosc.vm.fedcloud.eu`
  must be replaced with your DNS hostname.
* [minio.yaml](./minio/minio.yaml): `pangeo-eosc-minio.vm.fedcloud.eu` and
  `pangeo-eosc-minioapi.vm.fedcloud.eu` must be replaced with your DNS hostname.
  Replace `pvc-minio-c-scale` with the corresponding
  [PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
  available on your baremetal k8s cluster.
* [minio-secrets.yaml](./minio/minio-secrets.yaml): `rootUser` and
  `rootPassword` must be replaced with your own. Please note that they need
  to be at least 3 and 8-character long, respectively.

## Steps

Get a copy of this repository:

```bash
cd working/dir/
git clone git@github.com:pangeo-data/pangeo-eosc.git
cd providers/kubernetes/
```

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
