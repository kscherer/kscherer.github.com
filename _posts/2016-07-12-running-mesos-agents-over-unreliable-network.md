---
layout: post
title: "Running mesos agents over an unreliable network"
category: Mesos
tags: [linux, mesos, networking]
---

I have mesos agents located in three datacenters with a usually
reliable WAN connection. Occasionally though all the running tasks in
a DC get killed and it gets traced back to a WAN connection
interruption.

This hasn't been a big problem until recently when a fail over link
had high enough latency that the agents would disconnect and kill all
running tasks approx every half hour for about 12 hours. I tried to
figure out which configuration options need to be tweaked for the
master and agents to wait longer before killing tasks and this is what
I came up with.

Current setup:

Three DC: DC1 (central), DC2 and DC3.
Mesos 0.27.2 with custom python scheduler
3 node Zookeeper 3.4.5 cluster in DC1 with 3 HA mesos masters.
Zookeeper observer nodes in DC2 and DC3
Agents in DC2 connect to Zookeeper observer in DC2

From my research there are several timeouts that are at play here:

1) Zookeeper `ticktime` and `synclimit`. Unfortunately the zookeeper
[read-only observer feature][1] is not available yet, so when the
observer loses connection it drops connections to the agents. There
isn't an agent `zk_session_timeout` configuration option, but it looks
like the agent force expires the zk session after 10 sec (the master
default). If zk reconnects in less than 10sec the session still
expires, but the master is detected and everything works.

2) Mesos master `agent_ping_timeout` and `max_agent_ping_timeout`. The
master shuts down the agent after this timeout (75 sec by
default). This causes the slave to restart and kill all running tasks.

3) `agent_reregister_timeout` and `max_agent_reregister`_timeout. If there
was a master failover during a WAN outage, then this timeout may be
triggered. But the default is 10min so that shouldn't be a problem.

Here are my conclusions for my setup. Please let me know if I missed anything.

1) Since the ZK observers in DC2 and DC3 do not affect main ZK cluster
when disconnected, changing `ticktime` or `synclimit` is not necessary.

2) Increase `max_agent_ping_timeout` on masters so that
`(agent_ping_timeout * max_agent_ping_timeout)` is longer than most
WAN outages. In my case most outages are less than 10 mins so I am
trying `max_agent_ping_timeout` = 40. This means I do not need to
increase reregister timeout. Unfortunately `max_agent_ping_timeout` is
a global configuration and I cannot set this value differently for
agents in the different DCs.

[1]: https://issues.apache.org/jira/browse/ZOOKEEPER-1607
