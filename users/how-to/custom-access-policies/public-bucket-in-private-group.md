# Examples of custom access policies

This file contains examples of policies that might be useful for others.

## Example 1: create a read-only bucket for everyone but read-write for a group

Below are group- and bucket-policies for a group called `groupname`.

NB: `groupname` must be created in EGI Check-in by VO managers.

### Group policy

The group policy must have this name:

```
urn:mace:egi.eu:group:vo.pangeo.eu:groupname:role=member#aai.egi.eu
```

The group policy must be defined as follows:

```json
{
    "Version": "2012-10-17", 
    "Statement": [ 
        {
            "Effect": "Allow", 
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::groupname-*" 
            ]
        }
    ]
}
```

NB: The group policy can only be configured by the administrator of the MinIO service.

### Bucket policy

Below is the corresponding policy to limit read-only access to the bucket:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "*"
                ]
            },
            "Action": [
                "s3:GetBucketLocation",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::groupname-bucketname"
            ]
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "*"
                ]
            },
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::groupname-bucketname/*"
            ]
        }
    ]
}
```
