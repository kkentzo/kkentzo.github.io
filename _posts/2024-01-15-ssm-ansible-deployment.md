---
title: No-ssh deployment to EC2 using ansible and AWS Systems Manager
layout: post
tags:
    - aws
    - ec2
    - ssm
    - systems manager
    - ansible
    - golang
    - devops
keywords:
    - aws systems manager
    - deployment
    - ec2
    - golang
    - release
    - ansible

---

In a [previous post]({% post_url 2020-03-25-deploying-with-ansible-systemd %}), we
saw how to deploy an application (small golang service) using ansible
and systemd. In that flow, ansible execution depended upon the remote
server accepting ssh connections. However, there are a lot of
situations in which the remote server does not have an open ssh port
due to security reasons (e.g. compliance to security requirements).

In such cases, where there is not direct access to the EC2 instance,
we have the option of using [AWS Systems
Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html)
as a route of sorts to our remote host. AWS Systems Manager (SSM in
short) enables a multitude of capabilities on a fleet of "managed
nodes". In our case, a managed node is an EC2 instance that runs the
SSM agent and has the necessary IAM permissions for being part of
SSM's fleet of managed nodes. We will use SSM's ability to send a
command to a managed node (our EC2 instance). SSM's ["Run
Command"](https://docs.aws.amazon.com/systems-manager/latest/userguide/run-command.html)
functionality offers a variety of presets (called
[documents](https://docs.aws.amazon.com/systems-manager/latest/userguide/documents.html)
for various flows, including one that specifies how to execute an
ansible playbook locally (which is the one we will use).

In this post, we will build a solution step-by-step that will:

- prepare the AWS stack with all the necessary resources (tool: cloudformation)
- perform the deployment of the application and the dependencies (tools: ansible & AWS Systems Manager)

The full source code of the solution is hosted in [this
repository](https://github.com/kkentzo/deployment-ansible-ssm-systemd-demo).

## Prerequisites

This guide depends on the existence of the following tools:

- [`ork`](https://github.com/kkentzo/ork/releases/tag/v1.7.3): a workflow automation tool which we will use in order to define the various actions that have to be performed, their dependencies and their content
- [`aws-cli`](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html): the official cli tool for interacting with the AWS APIs
- [Session manager plug-in](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) for `aws-cli` (optional): open interactive sessions to EC2 servers using AWS Systems Manager

The presence of `ansible` on the dev machine is not necessary since
`ansible` will actually be executed on the remote server.

## The application

The application that we are going to deploy is a trivial http server
in `golang` that just returns a greeting along with an http status of
`200` (file `demo.go`):

```go
package main

import (
	"flag"
	"fmt"
	"log"
	"net/http"
)

func main() {
	var port string

	flag.StringVar(&port, "port", "8080", "Specify the service port")
	flag.Parse()

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		var name string
		if name = r.URL.Path[1:]; name == "" {
			name = "stranger"
		}
		fmt.Fprintf(w, "hello %s!", name)
	})
	log.Printf("Starting service [port=%s]", port)
	log.Fatal(http.ListenAndServe(fmt.Sprintf(":%s", port), nil))
}
```

## Solution Overview

Our solution is based on the approach that the ansible playbook will
be executed locally on our EC2 server (since there is no ssh
connection to the remote host). AWS SSM will be responsible for
downloading the ansible playbook to the server and executing it. For
that to happen we will need to send the relevant command to SSM over
the AWS API. This command needs the following pieces of information
which we will model in the form of environment variables to a bash
script containing the command:

- `INSTANCE_ID`: the id of the ec2 instance in which the command needs to be sent
- `ANSIBLE_PLAYBOOKS_PATH`: a link to a zip file in S3 containing the ansible playbooks
- `PLAYBOOK_FILE`: the playbook file to be executed
- `LOG_GROUP`: the AWS Log Group to which the logs of the ansible execution will be sent
- `AWS_REGION`: the AWS region to which we want to send the SSM command


The command script essentially checks that these variables are all
defined and subsequently [dispatches the SSM
command](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ssm/send-command.html)
(`scripts/ssm_send_command.sh`):

```bash
#!/bin/sh
# This script sends a command to AWS SSM that:
# - instructs a particular instance ($INSTANCE_ID)
# - to execute an ansible playbook ($PLAYBOOK_FILE)
# - that is located in an S3 bucket ($ANSIBLE_PLAYBOOKS_PATH)
# - and write the logs to a log group ($LOG_GROUP)
# - the command will be executed in a specific AWS region ($AWS_REGION)
# all these environment variables need to present for the script to run
# the output of the command can be inspected using aws cli as follows:
# $ aws logs tail $LOG_GROUP --follow

# stop script on command error
set -e

# do we have everything that we need?
[ -z "${INSTANCE_ID}" ] && { echo "INSTANCE_ID is missing"; exit 1; }
[ -z "${ANSIBLE_PLAYBOOKS_PATH}" ] && { echo "ANSIBLE_PLAYBOOKS_PATH is missing"; exit 1; }
[ -z "${PLAYBOOK_FILE}" ] && { echo "PLAYBOOK_FILE is missing"; exit 1; }
[ -z "${LOG_GROUP}" ] && { echo "LOG_GROUP is missing"; exit 1; }
[ -z "${AWS_REGION}" ] && { echo "AWS_REGION is missing"; exit 1; }

# run the command
# we use interpolation within single quotes: https://unix.stackexchange.com/a/447974
aws ssm send-command --document-name "AWS-ApplyAnsiblePlaybooks" --document-version "1" \
    --targets '[{"Key":"InstanceIds","Values":["'"${INSTANCE_ID}"'"]}]' \
    --parameters '{"SourceType":["S3"],"SourceInfo":["{\"path\": \"'"${ANSIBLE_PLAYBOOKS_PATH}"'\"}"],"InstallDependencies":["True"],"PlaybookFile":["'"${PLAYBOOK_FILE}"'"],"ExtraVariables":["SSM=True"],"Check":["False"],"TimeoutSeconds":["3600"]}' \
    --timeout-seconds 600 --max-concurrency "50" --max-errors "0" \
    --cloud-watch-output-config '{"CloudWatchOutputEnabled":true,"CloudWatchLogGroupName":"'"${LOG_GROUP}"'"}' \
    --region "${AWS_REGION}"

echo "Command was sent. Monitor using:"
echo "aws logs tail ${LOG_GROUP} --follow"
```

The dispatched SSM command will be received by the specified EC2
instance (which must be registered as an SSM node) and will be
executed locally in the server instance. Among other actions, the
playbook will download the release binary from the corresponding s3
bucket and install it locally on the EC2 instance as a systemd
service.

In general, the workflow is split in two parts. The first part runs on
our local machine / dev laptop (or alternatively in some CI process)
with the objective of uploading the necessary artifacts to S3 and
sending the SSM command and the second part runs on the EC2 server
consists solely of the ansible playbook execution.

More specifically, the workflow steps are:

1. [laptop] Deploy the ansible playbooks to s3
2. [laptop] build the application and upload the binary to s3
3. [laptop] Send the command to SSM
4. SSM sends the command to EC2
5. [ec2] Download and execute (locally) the ansible playbook from s3
6. [ec2] Download the binary from s3 (part of ansible playbook)

![Workflow diagram](/assets/ssm_ansible_workflow.png)

From the above, it is clear that there's some amount of preparatory
work to be done before that flow can be executed. Before we send the
SSM command, will need to ensure that:

- the EC2 instance is set up as an SSM managed node
- the ansible playbooks are zipped and uploaded to a specific S3 bucket (to which the EC2 instance has access)
- the application binary is uploaded to a specific S3 bucket (to which the EC2 instance has access)

Let's now do that work.

### Creating the AWS resources

For the purpose of this post, we will assume that the EC2 instance
already exists, has the appropriate security group and has a public
elastic IP.

The existing instance must also have the [SSM agent installed and
running](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html)
(empirically speaking, most EC2 linux images have the agent
pre-installed and enabled).

We will create an AWS stack with the following resources:

- an S3 bucket `ssm-demo-release-artifacts` that will host:
  - the zipped ansible folder with the playbooks and roles
  - the application binary to be deployed on our server
- an AWS instance role that:
  - allows the instance to serve as an SSM managed node
  - allows access to CloudWatch Logs (ansible logs will be sent there)
  - allows access to the S3 bucket with the release artifacts
- an AWS Cloudwatch Log Group to collect the SSM logs (ansible output)

We will express the above in a cloudformation template (`cloudformation/demo.yml`):

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Description: >-
  Provision the necessary resources for enabling ansible deployment over SSM

Parameters:
  ReleaseArtifactsBucketName:
    Type: String
    Description: The name of the release artifacts s3 bucket
  LogGroupName:
    Type: String
    Description: The name of the log group for the ansible execution logs

Resources:
  # ======================================
  # === Ansible / Deployment Resources ===
  # ======================================

  # SSM will write the command logs to this log group
  AnsibleLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Ref LogGroupName
      RetentionInDays: 30

  # S3 Bucket for release artifacts
  ReleaseArtifactsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref ReleaseArtifactsBucketName
      PublicAccessBlockConfiguration:
        BlockPublicPolicy: true
        BlockPublicAcls: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  # policy for accessing the release artifacts
  ReleaseArtifactsBucketPolicy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: "ssm-demo-release-artifacts-access"
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          -
            Effect: "Allow"
            Action:
              - s3:ListBucket
              - s3:GetObject
            Resource:
              - !Sub "arn:aws:s3:::${ReleaseArtifactsBucketName}"
              - "arn:aws:s3:::${ReleaseArtifactsBucketName}/*"
      Roles:
        - !Ref ServerRole

  # Server Role
  ServerRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: "ssm-demo-server-role"
      ManagedPolicyArns:
        # enable the instance to serve as an SSM managed node
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
        # provide access to Cloudwatch logs (for ansible deployment over SSM)
        - arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          -
            Effect: "Allow"
            Principal:
              Service:
                - "ec2.amazonaws.com"
            Action:
              - "sts:AssumeRole"

  # Instance Profile -- this must be attached to the EC2 instance
  ServerProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      InstanceProfileName: "ssm-demo-server-instance-profile"
      Roles:
        - !Ref ServerRole
```

Once the above cloudformation stack is deployed, we will need to
attach the instance profile that was created
(`ssm-demo-server-instance-profile`) to the existing EC2 instance
[either through the web console or using the CLI
tool](https://repost.aws/knowledge-center/attach-replace-ec2-instance-profile).

Our EC2 should now (hopefully) be visible under Systems Manager's
Fleet Manager (AWS Web Console).

### Deploying the application

Having set the necessary AWS resources, we will now focus on the
ansible playbook that will be executed on the remote host (EC2). We
will follow the pattern established the [previous post]({% post_url
2020-03-25-deploying-with-ansible-systemd %}) which installs the
application as a `systemd` service. The difference in the approach is
that we no longer send the binary over ssh but, rather, we first copy
the binary to the s3 bucket (`ssm-demo-release-artifacts`) and then
download the binary from within the ec2 server using ansible.

The relevant ansible task makes use of the aws cli tool (which must
exist on the server) and looks like so:

```yaml

- name: Download artifact to server
  ansible.builtin.shell: |
    aws s3 cp {{ release_binary_s3_path }} /usr/local/bin/demo
    chown demo:demo /usr/local/bin/demo
    chmod u+x /usr/local/bin/demo
  notify:
    - Restart demo service

```

The `demo` user and group are created in the rest of the ansible
playbook which can be found in its entirety in this
[repository](https://github.com/kkentzo/deployment-ansible-ssm-systemd-demo).

The port in which our demo service will bind is configurable in
`ansible/demo.yml` (ansible variable `demo_app_port`).

### Bringing it all together

Having discussed all the pieces of the solution, we will now automate
the relevant workflows using `ork` and express the flow in the form of
`Orkfile` tasks (more details [here](https://github.com/kkentzo/ork)).

As we saw previously, there are two main workflows:

- the management (creation/update) of the relevant AWS resources
- the deployment / release of the demo application

We will express the the management of the AWS resources by defining 3
ork tasks: one for creating the CF stack for the first time
(`cloudformation.create`), one for describing its status
(`cloudformation.describe`) and one for applying updates
(`cloudformation.update`). These tasks will make use of the
corresponding `aws-cli` functionality:

```yaml
global:
  env:
    - AWS_REGION: eu-central-1
      RELEASE_ARTIFACTS_BUCKET: ssm-demo-release-artifacts
      ANSIBLE_LOG_GROUP: "/ssm/ansible/demo"

tasks:
  - name: cloudformation
    env:
      - STACK_TEMPLATE: cloudformation/demo.yml
        STACK_NAME: demo-ansible-ssm
    tasks:
      - name: create
        description: create the cloudformation stack for the first time
        actions:
          - >-
            aws cloudformation create-stack
            --region ${AWS_REGION}
            --stack-name ${STACK_NAME}
            --template-body "file://${STACK_TEMPLATE}"
            --capabilities CAPABILITY_NAMED_IAM
            -- parameters
            ParameterKey=ReleaseArtifactsBucketName,ParameterValue=${RELEASE_ARTIFACTS_BUCKET}
            ParameterKey=LogGroupName,ParameterValue=${ANSIBLE_LOG_GROUP}

      - name: describe
        description: show the current status of the cloudformation stack
        actions:
          - aws cloudformation describe-stacks --stack-name ${STACK_NAME}

      - name: update
        description: apply changes to the cloudformation stack
        actions:
          - >-
            aws cloudformation update-stack
            --region ${AWS_REGION}
            --stack-name ${STACK_NAME}
            --template-body "file://${STACK_TEMPLATE}"
            --capabilities CAPABILITY_NAMED_IAM
            -- parameters
            ParameterKey=ReleaseArtifactsBucketName,ParameterValue=${RELEASE_ARTIFACTS_BUCKET}
            ParameterKey=LogGroupName,ParameterValue=${ANSIBLE_LOG_GROUP}
```

We can create the stack by running `ork cloudformation.create`; the
necessary AWS credentials must be properly set up in the shell before
we execute this command.

Having created these resources, we must now associate our
(pre-existing) EC2 instance with the instance profile
(`ssm-demo-server-instance-profile`) by modifying the instance's IAM
role (e.g. from the web console). We should also verify that our
instance is indeed visible under AWS System Manager's Managed Node
Fleet (it may take a while for the instance to appear under the
fleet).

We are now ready to deploy and release our application by:

- building the application (ork task: `build`)
- uploading our ansible playbook to s3 (ork task: `ansible.deploy`)
- upload the binary to S3 and trigger the SSM send command (ork task: `release`)

Here are the definitions of those tasks in the `Orkfile` (we need to
replace the `INSTANCE_ID` variable in the `Orkfile` with the actual ID
of our EC2 instance):

```yaml
global:
  env:
    - AWS_REGION: eu-central-1
      RELEASE_ARTIFACTS_BUCKET: ssm-demo-release-artifacts
      ANSIBLE_LOG_GROUP: "/ssm/ansible/demo"

tasks:
  - name: build
    description: build the demo binary
    env:
      - GOOS: linux
        GOARCH: amd64
    actions:
      - go build -o bin/demo app/demo.go

  - name: ansible.deploy
    description: deploy the ansible playbooks to AWS S3
    actions:
      - zip -r -FS ansible.zip ansible
      - aws s3 cp ansible.zip https://${RELEASE_ARTIFACTS_BUCKET}.s3.${AWS_REGION}.amazonaws.com/ansible.zip

  - name: release
    description: release the demo application over SSM
    env:
      - INSTANCE_ID: i-REPLACE_ME_WITH_ACTUAL_INSTANCE_ID
        ANSIBLE_PLAYBOOKS_PATH: https://${RELEASE_ARTIFACTS_BUCKET}.s3.${AWS_REGION}.amazonaws.com/ansible.zip
        PLAYBOOK_FILE: ansible/demo.yml
    depends_on:
      - build
      - ansible.deploy
    actions:
      - aws s3 cp bin/demo s3://${RELEASE_ARTIFACTS_BUCKET}/demo
      - ./scripts/ssm_send_command.sh
```

Tasks `build` and `ansible.deploy` are expressed as dependencies of
task `release` (see `depends_on` attribute, so it suffices to run `ork
release` in order to release the application.

It is worth repeating that the ansible playbook will be executed in
the server, so, once the SSM command is sent to AWS, its progress can
be inspected by tailing the corresponding Cloudwatch Log Group like
so:

```bash
$ aws logs tail /ssm/ansible/demo --follow
```

Once the playbook is finished, we should be able to perform an http
request to the service (depending also on the EC2's public IP,
security group etc. which are out of scope in this guide).

## Summary

We have seen how to deploy an application to an AWS EC2 instance not
by going over ssh but by utilizing AWS Systems Manager bringing a lot
of security-related advantages (IAM authorization, command auditing
etc.). AWS SSM has a lot more features than the Run Command that we
used in this guide and it is worth looking over [the
documentation](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html).

This flow can be applied to any application (i.e. not just golang)
that can be packaged in an archive and transferred to EC2 via S3 with
the necessary adjustments in the ansible playbook that is responsible
for the deployment and release of the application on the EC2 instance.

The source code files that were used in this guide (`Orkfile`,
cloudformation template, `ssm_send_command` script and the ansible
playbook) can be found in [this
repository](https://github.com/kkentzo/deployment-ansible-ssm-systemd-demo).

Hope you enjoyed this!
