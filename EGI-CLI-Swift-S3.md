# Using fedcloud and openstack clients to access CESNET object storage

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

## Use Openstack without fedcloudclient to access a specific project storage

It is possible to use `openstack` command through `fedcloudclient` to interact with
Openstack object storage, as [documented here](https://docs.egi.eu/users/data/storage/object-storage/#access-via-rclone).

However, this works well when there is only one project associated with a 
Virtual Organization (i.e. a one to one mapping between the two).
We currently have one Virtual Organization (vo.pangeo.eu) and two OpenStack projects associated with it.
We have created _vo.pangeo.eu-swift_, a new, separate OpenStack project to allow normal users 
(i.e. non admin users) of the _vo.pangeo.eu_ VO to work with an object store.
This way, we should be able to set more fine grained authorizations on buckets.
That's why we need to use `openstack` commands in replacement of the `fedcloudclient`.

Please configure these environment variables:
```
export OS_AUTH_URL=https://identity.cloud.muni.cz/v3
export OS_AUTH_TYPE=v3oidcaccesstoken
export OS_PROTOCOL=openid
export OS_IDENTITY_PROVIDER=egi.eu
export OS_ACCESS_TOKEN=$OIDC_ACCESS_TOKEN
export OS_PROJECT_ID=57102d3e06b7476088fe4924370ae170
export OS_STORAGE_URL=https://object-store.cloud.muni.cz/swift/v1
```

Then the following command should work:
```
openstack container list
```

## Retrieve Openstack token for Swift

```
# get OS_AUTH_TOKEN
$ openstack token issue -c id -f value
```

You'll need `OS_AUTH_TOKEN` and `OS_STORAGE_URL` in order to interact with Swift using Zarr.

## Retrieve S3 credentials

CESNET provides the following self-service to get S3 credentials:
https://docs.cloud.muni.cz/cloud/advanced-features/#s3-credentials.

Please run:
```
openstack ec2 credentials create
openstack ec2 credentials list
```

Once you've created a credential, you can retrieve it with the `list` command. Do not use create if you already have one. 

This will provide `access` and `secret` keys. The `endpoint` URL is: `https://object-store.cloud.muni.cz/`.

__Be really careful of what you do with your credentials, e.g. avoid living them into notebooks.__
