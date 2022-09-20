# How to upload data from an HPC center to pangeo-eosc object storage using EGI check-in


Basically follow the documentation [EGI-CLI-Swift-S3.md](./EGI-CLI-Swift-S3.md).


First, create an _egi_ environment using micromamba or any conda like tool.  

```
micromamba create -n egi  python jq s3fs awscli --yes  -c conda-forge
micromamba activate egi
pip install fedcloudclient
```

## Connect your environment with EGI Check-In

If your access token is not created yet, create it at [https://aai.egi.eu/token/](https://aai.egi.eu/token/).

It is a very long string, but do not worry, copy and past it instead of `<your_token>` (without any space beofore or after the `=` sign) like below:


```
export OIDC_ACCESS_TOKEN=<your_token>
fedcloud token check
```

If the last command returns something similar to

```
Token is valid until 2022-09-19 20:46:35 UTC
Token expires in 3323 seconds
```

you are good, properly identified and connected with EGI.  


## Connect your environment with pangeo-eosc object storage. 

You need a pair of Access and Secret keys (`aws_access_key_id` and `aws_secret_access_key`)
in order to have read-write access to pangeo-eosc object store space from your enviroment
through AWS S3 interface.  

Use the following command to see if you already have those keys or not:

``` 
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu ec2 credentials list
```

If you have nothing listed, then type the following command in order to create them:

```
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu ec2 credentials create 
```

Then list the credentials created by issuing the `credentials list` command above:

```
(egi) todaka@br146-050:~$ fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu ec2 credentials list
Site: CESNET-MCC, VO: vo.pangeo.eu, command: ec2 credentials list
+----------------------------------+----------------------------------+----------------------------------+------------------------------------------------------------------+
| Access                           | Secret                           | Project ID                       | User ID                                                          |
+----------------------------------+----------------------------------+----------------------------------+------------------------------------------------------------------+
| x1xx | x2xx | x3xxxxxx | x4xxxxxx |
+----------------------------------+----------------------------------+----------------------------------+------------------------------------------------------------------+

```

Copy the values at `x1xx` and `x2xx` and past them instead of `x1xx` and `x2xx` in the next command:

```
aws configure set aws_access_key_id x1xx
aws configure set aws_secret_access_key x2xx
``` 

Now, you just need to choose which _bucket_ or object _container_ you'll upload files.  

Here is how you can list your available buckets on pangeo-eosc object storage. 

```
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu container list
```

In this example, we chose `testfred` as destination bucket, pushing the `tar/` local folder to this bucket.

```
aws s3 sync tar/ s3://testfred/ --endpoint-url https://object-store.cloud.muni.cz
```

You can also create a new _container_ or _bucket_ using `container create` command, or through [Openstack Horizon web interface](https://dashboard.cloud.muni.cz/project/containers/).
