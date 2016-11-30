---
layout: post
title:  "Docker S3-Backed Private Registry On Every Host"
date:   2015-11-20 02:04:51
---

# **Update** - AWS has since released a [private registry service](/2015/12/23/aws-ec2-container-registry-is-better.html)

I attended [this talk][Talk] at a [GoLangPhilly][GoLangPhilly] meetup back in August where the speaker,
[Peter Shannon][pietrojs], detailed the setup his company has been using to run docker in production.  Every host there
runs a local docker registry server backed by S3.

That was the first I'd heard of this setup.  Usually posts discuss running a private registry in a dedicated spot
(usually on AWS, behind an ELB) and funneling all docker pulls and pushes through it.  Eliminating authentication issues
and a single point of failure (except s3 of course) seem like a big win with little downside.  I decided to try it out.

I realize tutorials for creating an S3-backed private registry are easily found on the web and the
[docs][docker registry configuration] are good, but I haven't seen a single version I'd follow beginning to end yet.
Here is the way we do it at [RealScout][RealScout].  Briefly, the steps covered involve:

1. Create a bucket
1. Grant bucket access
1. Run the registry container
1. Using the registry

In the end, we'll be able to run docker containers on EC2 and outside of EC2 in pretty much the same way, both using a
highly available S3-backed private registry.

## Create the bucket

I've given up on doing this with Cloudformation and just do this rare operation via the AWS console.

## Configure the bucket

Optional, but it's probably a good idea to turn on [server side encryption][SSE] for this bucket.  We use
[troposphere][tropogit] to generate templates.  [This gist][ssegist] generates:

{% highlight json %}
{
    "Resources": {
        "DockerRegistryBucket": {
            "Properties": {
                "Bucket": "docker-registry",
                "PolicyDocument": {
                    "Id": "DockerRegistryBucketPolicy",
                    "Statement": [
                        {
                            "Action": [
                                "s3:PutObject"
                            ],
                            "Condition": {
                                "StringNotEquals": {
                                    "s3:x-amz-server-side-encryption": [
                                        "AES256"
                                    ]
                                }
                            },
                            "Effect": "Deny",
                            "Principal": "*",
                            "Resource": [
                                "arn:aws:s3:::docker-registry/*"
                            ],
                            "Sid": "DenyUnEncryptedObjectUploads"
                        }
                    ],
                    "Version": "2012-10-17"
                }
            },
            "Type": "AWS::S3::BucketPolicy"
        }
    }
}
{% endhighlight %}

## Configure bucket access permissions

Access to the bucket backing the registry currently works two different ways in our environment, depending on whether
you're accessing from within EC2 or outside.  Within EC2, IAM roles can be utilized to control access.  Outside of EC2,
we use IAM user policies to manage access.  I hope to spend some time with [aws-mock-metadata][awsmockmetagit] or
[hologram][hologramgit] to eliminate this disparity.

The policy looks like this for the role and user, using [this gist][ssegist]:

{% highlight json %}
"Policies": [
	"PolicyName": "DockerRegistryConsumerPolicy",
	{
		"PolicyDocument": {
			"Statement": [
				{
					"Action": [
						"s3:ListBucket",
						"s3:GetBucketLocation"
					],
					"Effect": "Allow",
					"Resource": [
                        "arn:aws:s3:::docker-registry"
					]
				},
				{
                    "Action": [
                        "s3:GetObject"
					],
					"Effect": "Allow",
					"Resource": [
                        "arn:aws:s3:::docker-registry/*"
					]
				}
			]
		}
	}
]
{% endhighlight %}

This permits only pulls.  Add `s3:PutObject` for pushing.

## Run the registry container

Using this config, `dev-config.yml` in my case:
{% highlight yaml %}
---
version: 0.1
log:
  level: info
  formatter: text
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  s3:
    region: us-east-1
    bucket: docker-registry
    encrypt: true
    secure: true
    v4auth: true
    rootdirectory: /myroot
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
{% endhighlight %}

and this `docker-compose.yml`:
{% highlight yaml %}
---
registry:
  container_name: registry
  image: registry:2
  ports:
    - 5000:5000
  restart: always
  volumes:
    - ./dev-config.yml:/etc/docker/registry/config.yml:ro
{% endhighlight %}

You can `docker-compose up -d` on an EC2 box with the above policy attached to its IAM Role.

Outside of EC2, use a similar `docker-compose.yml` with credentials mounted in via `volumes`:
{% highlight yaml %}
- ${HOME}/.aws/docker-credentials:/${REGISTRY_USER}/.aws/credentials:ro
{% endhighlight %}

That credentials file follows the [standardized credentials][awscredstandard] file format.

If all is well, you should be able to list the available repos:

{% highlight console %}
$ curl -s http://localhost:5000/v2/_catalog
{"repositories":[]}
{% endhighlight %}

The docker docs are really quite good, [https://docs.docker.com/registry/introduction/][registry intro] is a great place
to continue on from here.

## Notes

We haven't hit any issues with this configuration yet but do worry a bit about eventual consistency issues with s3 that
a single registry server would address.

[GoLangPhilly]: http://www.meetup.com/GoLangPhilly
[RealScout]: http://realscout.com
[SSE]: http://docs.aws.amazon.com/AmazonS3/latest/dev/UsingServerSideEncryption.html
[Talk]: http://peterjshan.com/speaking/2015/08/golangphilly-phillydevops-combined-meetup/
[awscredstandard]: https://blogs.aws.amazon.com/security/post/Tx3D6U6WSFGOK2H/A-New-and-Standardized-Way-to-Manage-Credentials-in-the-AWS-SDKs
[awsmockmetagit]: https://github.com/dump247/aws-mock-metadata
[docker registry configuration]: https://docs.docker.com/registry/configuration/
[hologramgit]: https://github.com/AdRoll/hologram
[pietrojs]: https://github.com/pietrojs
[policygist]: https://gist.github.com/graphaelli/0d5cfb24c4255daab1a5
[registry intro]: https://docs.docker.com/registry/introduction/
[ssegist]: https://gist.github.com/graphaelli/3a1e43cb94b3e7e36ce5
[tropogit]: https://github.com/cloudtools/troposphere
