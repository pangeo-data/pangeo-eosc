# Using fedcloud and openstack clients to access CESNET object storage

## First install required Python packages

```
conda create -n egi python jq s3fs gcc awscli --yes -c conda-forge
conda activate egi
pip install fedcloudclient
```
See more information about `fedcloudclient` in: https://fedcloudclient.fedcloud.eu/. Installing `fedcloudclient` will also install required `openstackclient`.

You should be able to issue:
```
fedcloud token check
```

But you don't have a token yet.

You can also run `conda` from a container if don't want to install it. Steps are the same after the following command:
```
sudo docker run -it condaforge/mambaforge /bin/bash
```

## Get your access token from EGI Check-In

Go to https://aai.egi.eu/token/ to [obtain your access token](https://docs.egi.eu/users/aai/check-in/obtaining-tokens/token-portal/). Copy your access token, and then set it:

```
export OIDC_ACCESS_TOKEN=<your_token>
fedcloud token check
```

Last command should return a valid token.

## Use Openstack CLI via fedcloudclient

It is recommended to use `openstack` command through `fedcloudclient` to interact with
Openstack object storage, as [documented here](https://docs.egi.eu/users/data/storage/object-storage/#access-with-the-fedcloud-cli).

Depending on the workshop you are attending, the following command should work:
```bash
# CLIVAR workshop:
# https://www.clivar.org/events/arctic-processes-cmip6-bootcamp
fedcloud openstack --vo "/vo.pangeo.eu/swift" --site CESNET-MCC container list

# eScience workshop:
# https://www.aces.su.se/research/projects/escience-tools-in-climate-science-linking-observations-with-modelling/
fedcloud openstack --vo "/vo.pangeo.eu/escience" --site CESNET-MCC container list
```

Please note that the `--vo` option is supplied with different strings in the commands above depending on
the project that you are participating. The format is `--vo "/vo.pangeo.eu/<project-name>"`. If you are unsure
about your `<project-name>`, please [contact us](https://github.com/pangeo-data/pangeo-eosc/issues) and we will let you know.

## Retrieve Openstack token for Swift

```bash
# get OS_STORAGE_URL
fedcloud openstack --vo "/vo.pangeo.eu/escience" --site CESNET-MCC catalog show swift

# get OS_AUTH_TOKEN
fedcloud openstack --vo "/vo.pangeo.eu/escience" --site CESNET-MCC token issue -c id -f value
```

You'll need `OS_AUTH_TOKEN` and `OS_STORAGE_URL` in order to interact with Swift using Zarr.

## Retrieve S3 credentials

CESNET provides the following self-service to get S3 credentials:
https://docs.cloud.muni.cz/cloud/advanced-features/#s3-credentials.

Please run:
```
fedcloud openstack --vo "/vo.pangeo.eu/escience" --site CESNET-MCC ec2 credentials create
fedcloud openstack --vo "/vo.pangeo.eu/escience" --site CESNET-MCC ec2 credentials list
```

Once you've created a credential, you can retrieve it with the `list` command. Do not use create if you already have one. 

This will provide `access` and `secret` keys. The `endpoint` URL is: `https://object-store.cloud.muni.cz/`.

__Be really careful of what you do with your credentials, e.g. avoid living them into notebooks.__

In case, you need to re-set your creentials with following command. 

```
fedcloud openstack --vo "/vo.pangeo.eu/escience" --site CESNET-MCC ec2 credentials delete <`access` key> 
fedcloud openstack --vo "/vo.pangeo.eu/escience" --site CESNET-MCC ec2 credentials create
fedcloud openstack --vo "/vo.pangeo.eu/escience" --site CESNET-MCC ec2 credentials list
```
