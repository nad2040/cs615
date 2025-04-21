# Configuration Management

To scale changes across large numbers of systems.

Software Configuration Management systems - CMs

We cannot *just* blindly copy files from one system to another with rsync.

So, just store all the possible expected default configurations (golden image)
then deploy the shareable data
and make the host-specific changes

clear scalability issue

invert: let the machines pull the configuration from the one machine.
- problem: too much load on one server. (thundering herd problem)

even pub-sub model doesn't really work well.

Eventually, use a configuration management software like puppet.

however, you need to read the docs for several hours and look for dependencies and set up the server.

and then you run into an edge condition that isn't handled by the CM system.

site-specific global deployment requires different configuration per region
- e.g. resolv and syslog

configuration of systems may be divided into "base configuration" and "service definition"

Your servers have unique, yet predictable properties that vary based on workload
placement, specific purpose. E.g.,
• network configuration
• critical services such as DNS, NTP, or Syslog
• minimum OS / software version
• user management
• common service configuration (e.g., sshd(8))

## Syslog configuration example

Puppet example
```pp
class syslog (
  include cron
  include logrotate
  package {
    'syslogng" :
        ensure => latest,
        require => Service['syslogng'];
  }

service {
'syslogng' :
ensure = running,
enable = true; }
file (
/etc/syslogng/syslogng.conf:
ensure = file,
source => 'puppet:///syslog/syslogng.conf,
mode => '0644',
owner => 'root',
group => 'root',
require => Package['syslog-ng'],
notify = Service['syslog-ng'];
'/etc/logrotate.d/syslog-ng':
ensure => file,
source => 'puppet:///syslog/logrotate-syslogng',
mode => '0644',
owner => 'root',
group => 'root',
...
}
```

Chef cookbook
- ruby-like language
- for admin account

CFEngine
- defines ssh relevant bits

we abstract shared setup steps

- Review Variable vs. Static & Shareable vs. Non-Shareable Data - classify the
  common directories you might need to sync across machines accordingly.
- Identify a few common aspects of a service or a system and try to explicitly define its
  service description.
- Read up on Ansible, CFEngine, Chef, Puppet, and Saltstack. What do they have in
  common? How do they differ? How would you choose which one to use?
- How does Configuration Management relate to Infrastructure as Code or Service
  Orchestration?

# Part II

The objective of a CM system is NOT to make changes.
It should assert state.

There is a big state machine.
- unconfigured
- configured
- deviant
- unknown
- in service
- out of service

Distributed Systems - CAP Theorem:
- Consistency: all systems managed by the CM are consistent within their respective
  service definition.
- Availability: the services managed by the CM are kept available, even if no further
  updates or change sets can be retrieved.
- Partition tolerance: the CM system can (continue to) operate despite interruptions
  between its components; e.g. intermediate (coordinated) changes are not required.

We can only always guarantee 2 of these 3 properties.

CM systems assert state. all operations must be idempotent.
- the *result* must be the same. error messages don't really matter.

Convergence and Eventual Consistency

CM systems should ensure changes are:
- idempotent (on you)
- only applied if needed
- eventually consistent

This often requires complexity, coordination with and awareness of other systems.
Service Orchestration has developed as a separate, related discipline to help address
this.

- CM systems are complex themselves.
- CM systems are inherently trusted.
- CM systems can break everything. To the degree that you can't unbreak things
  afterwards.

Consider:
- staged rollout of change sets
- automated error detection and rollback
- self-healing properties
- authentication and privilege

CM Requirements:
- software installation
- service management / supervising
- file permissions / ownership
- static files
- host-specific data
- command-execution
- data collection

Your configuration management system provides or enables:
- a remote command execution agent
- a reporting agent
- a reporting infrastructure
- role-based actions and visibility

Overlapping information security related tasks:
- detection of deviation of known state
- integrity checks and intrusion detection
- patch management
- automated quarantine

Configuration Management overlaps with numerous other areas:
- software deployment (base OS, application packages, ...)
- monitoring (central reporting and ad-hoc data collection, ...)
- revision control and audit logs (CM changes are code changes!)
- compliance enforcement (e.g., baseline configurations)
- ...

Asset Inventory and Role Definitions
- lots of overlap

Containers:
- ought to be immutable
- But yes, at a mature organization the configuration of runtime systems has
  moved away from individual, mutable systems to generic, well-defined, immutable
  components that can be structured and deployed in an automated fashion: i.e.,
  Infrastructure as a Service.

Configuration management is NOT just for servers.
We need them on desktops, mobile clients, network equipment, storage devices, load balancers, ...

## Summary
- abstract services from the hosts or systems they run on
- move away from fragile systems managed by hand to reproducible, exchangeable instantiations of a service definition
- focus not on applying changes, but on state assertion
- as distributed systems, CM systems are subject to the CAP theorem
- CM systems require idempotence, state convergence with eventual consistence
- overlap with CI/CD, Deployment Engines, Service Orchestration, enforcement and monitoring agents
- CM underlies and enables immutable containers, Infrastructure as Code, laas

# Checkpoint

> Name an OS configuration management system:

Nix

> Do you have any experience using a configuration management system?

Not really. But I have installed nix-darwin for my Mac.

> What are some of the functional requirements for a configuration management system?

- software installation
- service management / supervising
- file permissions / ownership
- static files
- host-specific data
- command-execution
- data collection

> Give two example files under /etc that Configuration Management should control:

/etc/resolv.conf - dns configuration
/etc/syslog.conf - syslog configuration

> Review how your preferred package manager handles installation, deletion, and updates, especially with respect to configuration files. Are these operations idempotent?

My package manager is Homebrew. I think installation is not idempotent. It
triggers an update if outdated. Homebrew manages files with a bunch of
symlinks to the correct version.

> What protocol and port does syslog use?

UDP on port 514 (mainly)

> What other log mechanisms do you know? How do they differ from syslog?

I don't know other logging mechanisms. I guess modern Linux systems use
journalctl instead of syslog.

> On an EC2 instance of your choice, look at the files under /var/log.
> Identify one meaningful metric from one of those files that a system administrator would be interested in.
> Explain why, and how it would be useful.

The exact timestamp of an event. Not only does this make the ordering of events
clear, but also it allows the aspect of time to be used for analysis. If
something is taking a lot of time, the logs would make it obvious.

> What kind of tools can you use to process logs? Think about scalability!

awk(1). A parsing tool will be fast and massively parallel. Every application
might have a different log structure though. I'm not sure what the goal is for
log processing though.
