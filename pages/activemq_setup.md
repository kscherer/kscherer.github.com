---
layout: page
title: "Wind River ActiveMQ Infrastructure"
---

# {{ page.title }}

Here are some details of the Wind River ActiveMQ infrastructure that I
have installed to support Mcollective.  I have a non redundant
distributed setup: 1 activemq broker in each of 3 geographic locations
(master/slave is on the TODO list). 

Each broker is connected to every other with a TTL of 1. So far it
seems to be working. I recently upgraded to ActiveMQ 5.7.0 on CentOS
6.3 with openjdk 1.7. MCollective 2.2.x is setup with activemq
connector and thanks to recent MCollective mailing list info, now has
direct_addressing = 1 in the configuration. Note that if
direct_addressing is enabled in the client, but not on the server,
connections will fail.

I have put the spec file I used to make 5.6 and 5.7 activemq rpms on
[GitHub][activemq_repo]. I have also submitted this to puppetlabs, so hopefully it
will make it into puppetlabs repo.

Here are a few things I have learned.

Activemq 5.6+ tarball contains configurations which use a new
${activemq.data} variable. This variable needs to be setup in the
activemq-wrapper.conf. This is missing from the activemq 5.6 rpm in
the PE repo.

Snippets from default activemq.xml shipped by PuppetLabs

    <broker xmlns="http://activemq.apache.org/schema/core"
      brokerName="localhost" dataDirectory="${activemq.base}/data"
      destroyApplicationContextOnStop="true">
    </broker>

destroyApplicationContextOnStop is no longer supported in 5.6+.

    <transportConnectors>
      <transportConnector name="openwire" uri="tcp://0.0.0.0:61616"/>
      <transportConnector name="stomp+nio" uri="stomp+nio://0.0.0.0:61613"/>
    </transportConnectors>

The activemq stomp [example][2] has the following configuration.

    <transportConnector name="stomp+nio" uri="stomp://0.0.0.0:6163?transport.closeAsync=false"/>

I have added transport.closeAsync=false to my configs, but does anyone
know why this is a good idea for stomp? The activemq docs only states
that connections are closed synchronously.

I was also getting out of memory errors often. Turns out there is a
ActiveMQ FAQ [entry][3] for this:

After adding `-Dorg.apache.activemq.UseDedicatedTaskRunner=false` to the
activemq-wrapper.conf, ActiveMQ uses a lot less memory. Which probably
is why it seems to be much more stable now. Turning off the task
runner switches ActiveMQ to use a thread pool instead of a thread per
connection. Some [people][4] suggest that the thread pool should be the
default. It helped me, so I am passing this info along. I am curious
if anyone else has experience with this setting.

For the curious, the mess that is my puppet repos is also available on
[GitHub][5]. The templates for my activemq.xml, activemq-wrapper.conf
are located in modules/wr/templates.

[activemq_repo]: https://github.com/kscherer/activemq-rpm
[2]: http://svn.apache.org/repos/asf/activemq/trunk/assembly/src/sample-conf/activemq-stomp.xml
[3]: http://activemq.apache.org/javalangoutofmemory.html
[4]: http://www.commonsensecode.com/2010/07/02/activemq-supporting-thousands-of-concurrent-connections/
[5]: https://github.com/kscherer/puppet-modules 
