---
layout: page
title: "Puppet Infrastructure"
---

# {{ page.title }}

The Wind River Linux team has 3 major development centers in Ottawa,
Alameda and Beijing. These sites are connected by a fast WAN but I
decided that each site should have its own infrastructure of puppet
master, puppet dashboard, ActiveMQ node and mirrors.

## Workflow

The first problem is how to setup the puppet classes so that multiple
people can collaborate without causing problems. Luckily PuppetLabs
posted an article detailing a very good way to do this using
[Git and Puppet Environments][1]

I then scaled this out to three sites. I have one master git repo with
post-commit hooks to the three puppetmasters. Each puppet master then
has a commit hook to create the environment per branch.

The full workflow looks like this

    git checkout -b my_env production
    <edit>
    git commit -m "My new environment"
    git push

This pushes the change to the main puppet git repo. A post commit hook
immediately pushes the commit to a repo on each puppet master. On each
puppet master, a post commit hook runs and creates a new environment
my_env. To test the changes on a machine temporarily:

    puppet agent --test --environment my_env

For a more permanent change change the environment listed in the
puppet.conf. If the puppet.conf file is managed by puppet as it is at
Wind River, the puppet class will need to be modified to also use the
new environment.

When the change is tested, it can be merged back into the production
branch.

    git merge origin/my_env production

The --squash option can also be used to condense the history.

## MCollective

MCollective and ActiveMQ have always been designed to handle network
partitions. There is an ActiveMQ node in each site and the sites are
connected to form an ActiveMQ cluster. Each site also has its own
subcollective. This allows any mcollective client to access any
machine or restrict commands to a subset of machines. It is also
possible to restrict an ActiveMQ user (used when connecting to
ActiveMQ) to a subcollective.

[1]: http://puppetlabs.com/blog/git-workflow-and-puppet-environments/
