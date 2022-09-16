# Using fedcloud and openstack slients

## First install required Python packages

```
conda create -n egi python
pip install openstackclient
pip install fedcloudclient
```

You should be able to issue:
```
fedcloud token check
```

But you don't have a token yet.

## Connect to EGI

Go to https://aai.egi.eu/token/ to optain your client token. Copy your access token, and then set it:

```
export OIDC_ACCESS_TOKEN=<your_token>
fedcloud token check
```

Last command should return a valid token.

## Use Openstack

You need to set other env variables to simplify things:

```
export EGI_SITE=CESNET-MCC
export EGI_VO=vo.pangeo.eu
```

Then the following command should work:
```
fedcloud openstack container list
```

## Retrieve Openstack swift credentials

```
# explore sites with swift storage
$ fedcloud endpoint list --service-type org.openstack.swift --site ALL_SITES

# get OS_AUTH_URL
$ fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu catalog show keystone

# get OS_AUTH_TOKEN
$ fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu token issue \
  -c id \
  -f value

# get OS_STORAGE_URL for your site and Virtual Organisation
$ fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu catalog show swift
```

You'll need OS_AUTH_TOKEN and OS_STORAGE_URL in order to interact with Swift using Zarr.