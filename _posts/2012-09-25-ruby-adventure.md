---
layout: post
title: "Ruby adventure"
category: 
tags: []
---

Puppet is a Ruby project and many of the tools that work with Puppet
are also Ruby tools. For example, RSpec and Vagrant. To get access to
these tools, the "normal" path would be to use the Ubuntu package
manager, apt-get. But the Ruby world has its own packaging system,
rubygems. The first thing I tried and used was:

    gem install --user-install

which installs ruby code as a local user. Less use of
sudo and root access is a good thing. The only downside is adjusting
the PATH variable to find the rubygems.

The next trick is using RVM to install multiple ruby versions to user
account. This allows another level of containerization. Installation
is simple.

    curl -L https://get.rvm.io | bash -s stable

Only annoyance is that the script modifies my bashrc and
bash_profile. I erased those edits, sourced the rvm initialization
file and installed a recent ruby.

    source ~/.rvm/scripts/rvm
    rvm install 1.9.3
    rvm use 1.9.3

Next install the gem packages for testing inside the rvm

    gem install --no-ri --no-rdoc puppet rspec-puppet puppetlabs_spec_helper

Next step was to use rspec-puppet to run my puppet class unit tests,
but after much debugging, it turns out the move to Puppet 3.0.0 broke
rspec puppet.

    gem install puppet -v 2.7.19
    gem uninstall puppet -v 3.0.0

Now my rspec unit tests pass, but unfortunately just as slowly as
before. Looks like Ruby 1.9.3 didn't speed things up much.
