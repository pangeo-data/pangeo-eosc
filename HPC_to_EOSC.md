# How to upload data from HPC center to pangeo-eosc object storage using egi check-in


Basically follw the documentation [EGI-CLI-Swift-S3.md](./EGI-CLI-Swift-S3.md).


First, create egi enviroment using micromamba.  

```
micromamba create -n egi  python jq s3fs awscli --yes  -c conda-forge
micromamba activate egi
pip install fedcloudclient
```

## connect your enviroment with EGI Check-In
If your access token is not created yet, create it at [https://aai.egi.eu/token/](https://aai.egi.eu/token/)

It is very long, but do not afraid, copy and past it instead of `<your_token>` (without any space after the `=`) 


```
export OIDC_ACCESS_TOKEN=<your_token>
fedcloud token check
```

If this command returns you something similer to

```
Token is valid until 2022-09-19 20:46:35 UTC
Token expires in 3323 seconds
```
you are good.  


## connect your enviroment with pangeo-eosc object storage. 

We need 
aws_access_key_id and aws_secret_access_key 
to be able to have read-write access to pangeo-eosc disk space from your HPC enviroment.  

Use follwoing command to see if you already have aws_access_key_id and aws_secret_access_key or not.

``` 
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu ec2 credentials list
```

If you have nothing listed, then type
```
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu ec2 credentials create 
```

Then list the crednetials created witht the following list command.  

```
(egi) todaka@br146-050:~$ fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu ec2 credentials list
Site: CESNET-MCC, VO: vo.pangeo.eu, command: ec2 credentials list
+----------------------------------+----------------------------------+----------------------------------+------------------------------------------------------------------+
| Access                           | Secret                           | Project ID                       | User ID                                                          |
+----------------------------------+----------------------------------+----------------------------------+------------------------------------------------------------------+
| x1xx | x2xx | x3xxxxxx | x4xxxxxx |
+----------------------------------+----------------------------------+----------------------------------+------------------------------------------------------------------+

```

copy the value at x1xx and x2xx and past them instead of x1xx x2xx in next command 
```
aws configure set aws_access_key_id x1xx
aws configure set aws_secret_access_key x2xx
``` 

Now, you just need to chose which 'dataset_bucket' you'll upload file.  

Here is how you can list your available datset_bucket on pangeo-eosc object storage. 

```
fedcloud openstack --site CESNET-MCC --vo vo.pangeo.eu container list
```

In this example, we chose 'testfred' as dataset_bucket,

```
aws s3 sync tar/ s3://testfred/ --endpoint-url https://object-store.cloud.muni.cz
```

