+++
title = "AWS EC2 Container Registry is Better"
date = 2015-12-23T00:00:16-05:00
+++

I spent some of today with EC2 Container Registry following it's promotion to [GA][aws-ecr-ga] status and it's clear
that it's far better than what I [was using](/2015/11/20/docker-s3-private-registry.html).  For $0.07 more per
GB-month, you get the same features as any other private registry service:

- no need to run a local registry on every host
- global https access to a highly available registry
- registry browser UI

But you also get **IAM permissions** based access control!

The docs and AWS console walkthrough are great for getting started, the brief version is:

{{< highlight bash >}}
pip install -U awscli
aws ecr create-repository --repository-name <name>
$(aws ecr get-login --region us-east-1)
# docker tag, push ...
{{< /highlight >}}


## docker pull permissions

The policy looks like this for the role and user, using awacs and troposphere:

{{< highlight python >}}
# Permit AWS-hosted docker registry access
from awacs.aws import Allow, Everybody, Policy, Statement
import awacs.iam as iam
import troposphere.iam

troposphere.iam.Policy(
	PolicyName="DockerRegistryConsumerPolicy",
	PolicyDocument=Policy(
		Statement=[
			Statement(
				Effect=Allow,
				Action=[
					iam.Action("ecr", "BatchCheckLayerAvailability"),
					iam.Action("ecr", "BatchGetImage"),
					iam.Action("ecr", "DescribeRepositories"),
					iam.Action("ecr", "GetAuthorizationToken"),
					iam.Action("ecr", "GetDownloadUrlForLayer"),
					iam.Action("ecr", "GetRepositoryPolicy"),
					iam.Action("ecr", "ListImages"),
				],
				Resource=[Everybody],
			)
		]
	)
)
{{< /highlight >}}

and generates:

{{< highlight json >}}
"Policies": [
	"PolicyName": "DockerRegistryConsumerPolicy",
	{
		"PolicyDocument": {
			"Statement": [
				{
					"Action": [
						"ecr:BatchCheckLayerAvailability",
						"ecr:BatchGetImage",
						"ecr:DescribeRepositories",
						"ecr:GetAuthorizationToken",
						"ecr:GetDownloadUrlForLayer",
						"ecr:GetRepositoryPolicy",
						"ecr:ListImages"
					],
					"Effect": "Allow",
					"Resource": [
						"*"
					]
				}
			]
		}
	}
]
{{< /highlight >}}

This permits login, pulling images, and browsing the repositories.  Since we're using AWS, command line tools to
substitute for `docker search` are easily within reach

   * `aws ecr describe-repositories` to view the repos
   * `aws ecr list-images --repository-name <name>` to see image:tag availability.

## docker push permissions

So far, I've simply logged in as a registry administrator (`Allow`ed to `ecr:*`) when needing to push images -
eventually CI will be the only user allowed to assume that role.  For now that is:

`docker logout https://$acct.dkr.ecr.us-east-1.amazonaws.com`

followed by

`$(aws ecr get-login --region us-east-1 --profile registry-admin)`.

[aws-ecr-ga]: https://aws.amazon.com/blogs/aws/ec2-container-registry-now-generally-available/
