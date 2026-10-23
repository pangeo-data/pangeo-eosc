# How to deploy Pangeo using a baremetal kubernetes cluster

These are the steps to deploy [Jupyterhub](https://jupyter.org/hub) connected to [Dask](https://kubernetes.dask.org/) and [MinIO](https://min.io/) using the baremetal k8s cluster at [CESNET](https://www.egi.eu/partner/cesnet).

## Assumptions for deployment

This guide assumes that you have:

1. Access to a kubernetes cluster with `kubectl`.
2. [Helm](https://helm.sh/) is installed (i.e. the `helm` command works).
3. Access to configure DNS hostnames (e.g. the
   [EGI Dynamic DNS](https://docs.egi.eu/users/compute/cloud-compute/dynamic-dns/)).
4. Credentials for an OIDC (e.g. the
   [EGI Check-in](https://docs.egi.eu/providers/check-in/sp/)).
5. The kubernetes administrator has deployed the
   [Dask Kubernetes Operator](https://kubernetes.dask.org/).

## Get a copy of this repository

For example, using `git clone`, and make sure you are in the correct
workding directory:

```bash
cd working/dir/
git clone git@github.com:pangeo-data/pangeo-eosc.git
cd providers/kubernetes/
```

# JupyterHub connected to Dask

To deploy [Jupyterhub](https://jupyter.org/hub) we use the official documentation from [Zero to JupyterHub with Kubernetes](https://z2jh.jupyter.org/).

## Before going ahead

Please configure the values below accordingly:

* [jupyterhub-secrets.yaml](./jupyterhub/jupyterhub-secrets.yaml): replace `client_id`, `client_secret`, and `oauth_callback_url` with your own values.
* [jupyterhub.yaml](./jupyterhub/jupyterhub.yaml): `pangeo-eosc.vm.fedcloud.eu` must be replaced with your DNS hostname.
* [dask-user-role.yaml](./dask/dask-user-role.yaml) and [dask-user-rolebinding.yaml](./dask/dask-user-role.yaml): `c-scale-pangeo-dask` must be replaced with your own namespace.

## Deployment

Deploy JupyterHub with the `helm` command below:

```bash
helm upgrade jupyterhub jupyterhub/jupyterhub \
   --cleanup-on-fail \
   --wait --install \
   --create-namespace \
   --namespace c-scale-pangeo-dask \
   --values jupyterhub/jupyterhub.yaml \
   --values jupyterhub/jupyterhub-secrets.yaml
```
Note: `c-scale-pangeo-dask` must be replaced with your own namespace.

Then configure RBACs for Dask:

```bash
kubectl apply -f dask/dask-user-role.yaml
kubectl apply -f dask/dask-user-rolebinding.yaml
```
All going well JupyterHub will be available at [https://pangeo-eosc.vm.fedcloud.eu](https://pangeo-eosc.vm.fedcloud.eu).

See an [example notebook](../../users/how-to/dask-operator-test.ipynb) to get started.

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
helm upgrade minio bitnami/minio \
  --install --wait \
  --cleanup-on-fail \
  --create-namespace \
  --namespace c-scale-pangeo-dask \
  --version 16.0.10 \
  --values minio/minio.yaml \
  --values minio/minio-secrets.yaml
```

All going well the **MinIO Console** will be available at
[https://pangeo-eosc-minio.vm.fedcloud.eu](https://pangeo-eosc-minio.vm.fedcloud.eu)
and the **MinIO API endpoint** will be available at
[https://pangeo-eosc-minioapi.vm.fedcloud.eu](https://pangeo-eosc-minioapi.vm.fedcloud.eu).

See an [example notebook](../../users/how-to/object-storage-minio-test.ipynb) to get started.
