# Using fedcloud and openstack slients

## First install required Python packages

```
conda create -n egi python jq --yes
conda activate egi
pip install fedcloudclient
```
See more information about `fedcloudclient` in: https://fedcloudclient.fedcloud.eu/. Installing `fedcloudclient` will also install required `openstackclient`.

You should be able to issue:
```
fedcloud token check
```

But you don't have a token yet.

## Get your access token from EGI Check-In

Go to https://aai.egi.eu/token/ to [obtain your access token](https://docs.egi.eu/users/aai/check-in/obtaining-tokens/token-portal/). Copy your access token, and then set it:

```
export OIDC_ACCESS_TOKEN=<your_token>
fedcloud token check
```

Last command should return a valid token.

## Use Openstack


Then the following command should work:
```
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu container list
```

## Retrieve Openstack Swift credentials

```
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

OS_STORAGE_URL is actually always https://object-store.cloud.muni.cz/swift/v1 for CESNET. So what you really need above is OS_AUTH_TOKEN.


## Retrieve S3 credentials

CESNET provides the following self-service to get S3 credentials:
https://docs.cloud.muni.cz/cloud/advanced-features/#s3-credentials.

Using `fedcloudclient` you can do:
```
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu ec2 credentials create 
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu ec2 credentials list
```

Once you've created a credential, you can retrieve it with the `list` command. Do not use create if you already have one. 

This will provide `access` and `secret` keys. The `endpoint` URL is: https://object-store.cloud.muni.cz/.

__Be really careful of what you do with your credentials, e.g. avoid living them into notebooks.__