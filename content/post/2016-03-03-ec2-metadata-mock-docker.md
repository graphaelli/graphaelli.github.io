---
type: "post"
title: EC2 Metadata service mocking with Docker
---

My former [employer](http://monetate.com) quietly open sourced a handy utility for mocking out the metadata service on AWS' EC2
hosts a few weeks ago.  As I've mentioned [previously](/2015/11/20/docker-s3-private-registry.html), there are a few
projects in this space, but none quite as simple as this one to get going.

[ectou-metadata](https://github.com/monetate/ectou-metadata) includes an implementation of the IAM security credential API that, to applications, feels exactly like
running on an EC2 instance launched with an IAM Role.  This means no more injecting credentials into containers needing
AWS API access.  It also helps increase [dev prod parity](http://12factor.net/dev-prod-parity).

Here's how I'm using it out under Mac OSX with a stock `docker-machine` virtualbox VM.

First, build the docker container image.  I've contributed a [Dockerfile](https://github.com/monetate/ectou-metadata/pull/1).

Next, plumb the well-known address in the docker-machine VM:

{{< highlight bash >}}
docker-machine ssh <machine-label> sudo ip addr add 169.254.169.254 dev eth0
{{< /highlight >}}

Finally, launch a container bound to that IP.  This works with `--net` if using docker networking.

{{< highlight bash >}}
docker run \
  --env=MOCK_METADATA_ROLE_ARN=arn:aws:iam::ACCOUNT:role/ROLE \
  --publish=169.254.169.254:80:5000 \
  --volume=$HOME/.aws/mock-metadata-credentials:/home/ec2-user/.aws/credentials:ro \
  ectou-metadata
{{< /highlight >}}

If everything is wired up correctly you'll see 200s logged by the ectou-metadata service when making API calls, like:

{{< highlight bash >}}
172.17.0.1 - - [01/Feb/2016 18:54:29] "GET /latest/meta-data/iam/security-credentials/ HTTP/1.1" 200 9
172.17.0.1 - - [01/Feb/2016 18:54:29] "GET /latest/meta-data/iam/security-credentials/role-name HTTP/1.1" 200 641
{{< /highlight >}}

Other containers running on this VM that have network access will be able to use these credentials.

I'd love to figure out how to run multiple instances of this within a single VM so different containers automatically
have access to different roles but for now I've settled on one instance per VM/set of containers.  If there's anyone out
there, please [get in touch](/about) with thoughts.

There are some
other [goodies](https://github.com/monetate/ectou-metadata/blob/2ab1894c619e19b6f5062101bad24cd37ab32910/ectou_metadata/service.py#L22-L26)
baked in that look useful.  Big thanks to Monetate for opening up this code!
