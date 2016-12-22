---
layout: post
title: "Puppet Infrastructure Overhaul"
category: Puppet
tags: ["puppet"]
---

I have been planning to upgrade my infrastructure to Puppet 4 but
other priorities have delayed it. I was finally able to find a way to
start the upgrade work. There are many new pieces of technology
available which I hope will make things work even better than before.

### Puppet 4

Since Puppet 3 is End Of Life at the end of 2016, this upgrade is
probably the most urgent. I am looking forward to being able to use
the improved Puppet language and r10k. The Puppet Server is supposed
to be much faster and the AIO packages should be easier to install and
support.

### MCollective Choria

[R.I.Pienaar][12] has been busy and built a new mcollective deployment package
called [Choria][1]. It has puppet modules which automatically enables
SSL everywhere, has an audit plugin, a packager for plugins and uses
NATS instead of ActiveMQ. My federated cluster with three ActiveMQ
servers has been stable, but it was a pain to setup and
upgrade. It is also managed using a custom puppet module which I do
not want to maintain. I am also hoping to be able to use NATS as a
message bus for some application orchestration.

### Gitolite

I maintain a large internal network of git servers. The base
configuration is very open and anyone with a valid ssh login using NIS
can create or push to repositories. Every repository is available for
unauthenticated read-only access. We have a few post-receive hooks to
limit who can push to what repositories, but our developers respect
our gatekeeper model and do not push to repositories they aren't
supposed to. The open access model has allowed people to do emergency
fixes when necessary. But there has occasionally been requests for
some sort of access control and I also have considered locking down
the repository with the Puppet modules because it is so critical to
the business, so I decided to experiment with [Gitolite][2].

### R10K

I have been using [librarian-puppet][3] with a custom git
synchronization program which relies on the ActiveMQ network. I have 3
puppet masters and the post-receive hook uses STOMP to broadcast
changes. The [git-stomp-hook][4] receives the broadcast and calls
librarian-puppet as appropriate. This has worked well except for when
the ActiveMQ network was having problems. So I was happy to notice
that the [puppet-r10k][5] module contains a webhook program that can
be used to trigger r10k deploy on the puppet masters. Since r10k was
integrated into [Puppet Enterprise][6], I decided to move away from
librarian-puppet. R10k actually works very similarly to the solution I
had cobbled together, it just ignores module dependencies. This is
both a blessing and a curse, but because Puppet does not support
conditional dependencies it may be better long term to manage
dependencies manually.

### Bootstrapping a Puppet Server

I manage my Puppet 3 server using Puppet and the bootstrap process is
tricky. Given a machine with just the puppet agent, how to get the
Puppet Server + Hiera + R10K and my control repo installed in a
reproducible way. I started with the Puppetlabs [control-repo][7]
skeleton which gave me the basics, but no bootstrap. I looked through
a lot of repos and finally found [puppetinabox control-repo][8]
by [rnelson0][9]. This repo uses a script to install the bootstrap
modules locally and puppet apply with some simple puppet manifests to
do the bootstrap. I decided to use this approach as well.

### A Puppet module to manage Puppet

Next step was to choose a module to manage the Puppet server. I
reviewed many but many had crazy dependencies or didn't support the
way I wanted to configure my systems. I ended up using
the [puppet-puppet][10] module maintained by the [Foreman][11]
team. It is a big module, but it supports:

- Puppet agent run using cron
- Puppet server setup on Ubuntu 16.04
- Compatible with Puppetdb and r10k
- Foreman integration

I did add Foreman integration to my Puppet3 module, so having that was
interesting to me.

### Scripting the bootstrap

The bootstrap script does the following:

1. Make sure git is installed
2. Clone all the required modules into a bootstrap directory. I make
   internal git mirrors of all the puppet modules I use.
3. Run puppet apply using 3 manifests to install puppet server, hiera
   and r10k.
4. Run r10k deploy to generate the local production environment.

I now had a server setup and could start creating roles and profiles
to manage the server.

### R10K Webhook

Redundancy in infrastructure is good and having two ways to synchronize
the environments on the masters is also a good idea. I could use
mcollective, but that hasn't been setup yet. The [puppet-r10k][5]
module comes with a webhook. This webhook is a small ruby sinatra
application that listens for http connections and triggers r10k
commands as appropriate. It supports GitHub, GitLab, Bitbucket,
etc. but I don't need those. Since I am using a local gitolite server
I created a git post-receive hook that calls curl with the updated
branch:

    curl -d "{ \"ref\": \"$REFNAME\" }" -H "Accept: application/json" \
     "https://puppet:puppet@$HOST:8088/payload" -k -q

By default the webhook and r10k run as root which is something I try
to avoid. I was able to change the user for the webhook to puppet,
chown all the r10k cache and environment dirs to puppet user and
everything works. It also uses the SSL certs as signed by the Puppet
CA to encrypt the communication.

### Bash post receive hook and subshells

The only problem with this approach is that the user much wait when
running git push for the script to complete. I was able to run the
curl command in a subshell and have the post-receive script exit
quickly.

    ( trigger_webhook "$refname" <hostname> ) &

This code will run to completion even once the parent shell has
exited. The logs of the synchronization are stored on the puppet
master. They could be stored on the git server as well but that isn't
necessary.

### Next steps

Install MCollective Choria and start porting the base configuration
with ntp, ssh keys, package management, etc. to Puppet 4.

[1]: http://choria.io/
[2]: http://gitolite.com/
[3]: http://librarian-puppet.com/
[4]: https://github.com/kscherer/git-stomp-hooks
[5]: https://github.com/voxpupuli/puppet-r10k
[6]: https://puppet.com/product
[7]: https://github.com/puppetlabs/control-repo.git
[8]: https://github.com/puppetinabox/controlrepo
[9]: https://rnelson0.com/
[10]: https://github.com/theforeman/puppet-puppet
[11]: https://theforeman.org/
[12]: https://www.devco.net/
