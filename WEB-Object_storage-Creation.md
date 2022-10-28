# Creating object storage container/bucket from Openstack web interface provided by CESNET.

## Log on to openstack interface
Go to [Openstack dashboard of CESNET](https://dashboard.cloud.muni.cz/auth/login/) and chose EGI Check-in.

## Create containers/buckets
Click 'Object Store' -> 'Containers'

You will see the list of existing containers.  

Click '+Container', and in the pop up window labelled _Create Container_, define
**Container Name** and if the container should be **Public** or **Not public**.
**Not public** containers will only be accessible with Swift token or S3 credentials.
**Public** ones will be accessible on read only mode by everyone.

## Permissions

Please see below a table with the permissions to access object storage.

  | Create/destroy VMs | Object Storage at OpenStack: project "vo.pangeo.eu" | Object Storage at OpenStack: project "vo.pangeo.eu" | Object Storage at OpenStack: project "vo.pangeo.eu-swift" | Object Storage at OpenStack: project "vo.pangeo.eu-swift" | Object Storage at OpenStack: project "vo.pangeo.eu-escience" | Object Storage at OpenStack: project "vo.pangeo.eu-escience"
-- | -- | -- | -- | -- | -- | -- | --
Virtual Organisation |   | Public bucket | Private bucket | Public bucket | Private bucket | Public bucket | Private bucket
member of vo.pangeo.eu in aai.egi.eu/pangeo.admins | yes | read/write access | read/write access | read-write access | read/write access | read-only | no access
member of vo.pangeo.eu in aai.egi.eu | no | read-only | no access | read/write access | read/write access | read-only | no access
member of vo.pangeo.eu in aai.egi.eu/escience | no | read-only | no access | read-only | no access | read/write access | read/write access
member of vo.pangeo.eu in aai-dev.egi.eu | no | read-only | no access | read-only | no access | read-only | no access
None | no | read-only | no access | read-only | no access | read-only | no access

