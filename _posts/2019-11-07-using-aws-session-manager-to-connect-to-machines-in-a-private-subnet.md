---
layout: post
title: "Using AWS Session Manager to connect to machines in a private subnet"
category: AWS
tags: [linux, aws]
---

## Introduction

We are experimenting with AWS as many people are. One of the first
hurdles is connecting over SSH to the EC2 instances that have been
created. The "standard" mechanism is to setup a Bastion host that has
a restrictive "Security Group" (also known as Firewall). This Bastion
host is accessible from the Internet and once the user has logged into
this host they can then access other instances in the VPC.

The Bastion host has a few limitations:

- It is exposed to the Internet: A Security Group can restrict access
  to specific IPs and only open port 22. This is reasonably secure,
  but the possibility of an exploit in the SSH server is always a
  possibility.
- SSH key management: The AWS console allows for the creation of SSH
  keypairs that can be automatically installed on the instance which
  is great. If you have multiple people accessing the Bastion
  instance, then either everyone will have to use the same keypair
  (which is bad) or there needs to some other mechanism to managing
  the authorized_keys file on the Bastion instance. Ideally this is
  automated using a tool like Puppet Bolt or Ansible.

One of my weekly newsletters pointed me to [aws-gate][1] which mentioned
the possibility of logging into an instance using SSH without the need
for a Bastion host. This post documents my experience getting it
working.

## Local Requirements

On the local machine the AWS CLI must be installed. I use a python
virtualenv to keep the python environment separate and avoid requiring
root access.

    > python3 -m venv awscli
    > cd awscli
    > bin/pip3 install awscli

Unfortunately it turns out the Session Manager functionality requires
a special plugin which is only distributed as a deb package.

    > curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
    > sudo apt install ./session-manager-plugin.deb

The AWS CLI requires an access key. Go the AWS console -> "My Security
Credentials" and create a new Access key (or use existing
credentials).

    > ~/awscli/bin/aws configure
    AWS Access Key ID [None]: accesskey
    AWS Secret Access Key [None]: secretkey
    Default region name [None]: us-west-2
    Default output format [None]:

Also in the AWS EC2 console, create a new KeyPair and download the
.pem file locally. I put the file in ~/.ssh and gave it 0600
permissions. Now add the following to your .ssh/config file:

    # SSH over Session Manager
    host i-* mi-*
    ProxyCommand sh -c "~/awscli/bin/aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
    IdentityFile ~/.ssh/<keypair name>.pem

## AWS IAM Setup

By default an EC2 instance will not be manageable by the System
Manager. Go to AWS Console -> IAM -> Roles to update the roles.

I already had a default EC2 instance role and I had to add
`AmazonSSMManagedInstanceCore` permissions to the instance role.

## Launching the Instance

According to the docs the official Ubuntu 18.04 server AMI has the SSM
agent integrated and I relied on this. Finding the right AMI is really
frustrating because there aren't proper organization names attached to
AMIs. The simplest is to go the [Ubuntu AMI finder][2] and search for
'18.04 us-west-2 ebs' and select the most recent AMI.

In the launch options:

- choose the correct VPC with a private subnet
- the 'IAM Role' with the correct permissions
- A "Security Group" with port 22 open to you
- Select the Keypair that was downloaded earlier and setup in your
  .ssh/config file.

Launch the instance and wait a while. Go to the AWS Console -> Systems
Manager -> Inventory to see that the instance is running and the SSM
agent is working properly.

## Connecting over SSH

If everything is setup correctly grab the instance name and do the
login:

    > ssh ubuntu@i-014633b619400dfff
    Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-1052-aws x86_64)
    <snip>
    ubuntu@ip-10-0-1-193:~$

SSH access without a Bastion host is possible!

[1]: https://github.com/xen0l/aws-gate
[2]: https://cloud-images.ubuntu.com/locator/ec2/
