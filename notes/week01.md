# Introduction

System Administration is a broad field that

requires

breadth of knowledge:
- OS concepts
- TCP/IP concepts
- programming
- cloud computing
- ...

and depth of knowledge
- certain OS flavor
- specific services (DNS, email, databases, content-delivery)
- specific implementation/vendor (Oracle, Hadoop, Apache, Cisco)
- specific area of expertise (security, storage, network, datacenter)

---

practice
- how to ask questions
- where to ask questions
- read critically
- know what you don't know (dunning-kruger effect)
- understand what you're doing
- understand why you're doing it
- seek information exchange

# Job of a System Administrator

Containerized Applications

Computer Management
- server infra. "order to chaos"
- building design (Yahoo chicken coop design)

Access control
- biometrics
- security devices
- every device can have flaws

Connecting things

After WiFi, you lose control. It is still your job but other people could be the cause of your issues.
- like setting up a rogue access point

you have to provide services but people could be using your service for things you might be held liable for
- like criminal activity, etc.

systems administrators do a lot of typing
- ergonomic keyboard!

the Internet!
- managing server racks, ethernet switches, routers,
- web servers, databases, load balancers, firewalls, message queue
- access for administrators through devices (physical and remote!)
- 3rd party integrations!
- or infra in the cloud

Nobody will notice until things go wrong.

Physical tools can also be useful. You never know what can go wrong.

Every sysadmin has their own toolbox.

So what does a sysadmin actually do?
- no precise job description
- often learned by experience
- "makes things run"
- work behind the scenes
- often known as IT Support, Operator, Network Administrator, System Programmer, System Manager,
  Service Engineer, Site Reliability Engineer, etc.

Two newer names: DevOps and SRE

asking the dictionary
`dict <term>`

- A system is a "group of independent but interrelated elements comprising a unified whole"
- An administrator is "one who directs, manages, [...] or dispenses"
- A system administrator is a "person in charge of managing and maintaining a computer
  system of telecommunication system (as for a business or institution)"

key point is working on behalf of another

We will consider computer-human systems
- computer
- network
- users
- organization's goals

Core pillars of exceptional system design
- Scalability
- Security
- Simplicity

people think the internet is more just an abstract cloud but sysadmins know it's a duct-taped mess.

# Sysadmin Core Principles and Rules

Simplicity enables Scalability and Security.

Scalability:
- system overload from poor management.
- Solution 1: Scaling vertically (getting a bigger server) - easy to do - throws money at the problem - only buys you time
- Solution 2: Scaling horizontally (getting more servers) - distribute the load - load balancer - adds complexity
- Solution 3: Both 1 and 2 - higher costs!
- Remember you want to be able to scale down!

Elasticity - Being able to scale up and down

People often leave security as an afterthought, so it isn't suprising that people see
usability and security as inversely related. Instead we should see security as an enabling
factor.

It is important to differentiate Complicated and Complex
- Complex systems may be well organized
- Complicated systems are irregular and harder to follow

scalable and secure systems are less complex and much less complicated.

Keep it simple, stupid (KISS)

Ockham's Razor
- simplest explanation is probably correct

2nd law of thermodynamics - more entropy (more users)
- more systems -> more errors
- long running systems -> more errors

Hanlon's Razor
- never attribute to malice that which is adequately explained by stupidity
- preventing accidental failure actually helps prevent intentional abuse

Pareto's Priniciple
- most of the time, roughly 80% of the consequences derive from 20% of the causes

Sturgeon's Law
- 90% of everything is crap
- useful to set your expectations

Murphy's Law
- anything that can go wrong will go wrong
- anything that can happen, will

Causality
- for every effect, there must be a cause
- it will be hard to find the cause

Recommended exercises:
- try to find out details about how companies maintain their infrastructure
- try to find out what schools grant a degree on system administration
- determine the UNIX commands you use the most frequently and analyze the complexity and interfaces

# UNIX History
see cs631 week 01

# Git
how to check the ssh fingerprint
get the host information some other way!
`curl -s <link to info> | tail -1 >> ~/.ssh/known_hosts`

`git config --global --edit`
`git config --global push.default simple`

```sh
mkdir .cs615.git
cd .cs615.git
git init --bare
cd ..
git clone /path/to/.cs615.git cs615
cd cs615
cat >README <<EOF
...
EOF
git add ...
git commit -m ...
git push
^D
git clone hostname:.cs615.git cs615 # remote through ssh config!
```

# AWS EC2

`whois 155.246.0.0/16`

layers around the instance:
- subnet - allows you to assign IP space (RFC1918) 10/8 172.16/12 192.168/16
- security groups - can be used to allow/disallow traffic
- vpc - virtual private cloud, needs gateway

Had to create a user!!!
Then had to create access key.
Then had to do `aws configure` with the Access Key ID and secret access key
and region name and output format.

---

# Checkpoint

Describe in one, two, or at most three simple sentences the job of a System Administrator.

> The job of a system administrator is to help an institution manage its computer systems with
> whatever tools necessary.

What specific skills do you hope to learn in this course?

> I hope to ingrain some common shell command usages into my muscle memory so I don't have to keep
> checking the man pages or run tldr. I just want to know them by heart. If I make any progress
> towards this goal, I'll be happy.

What is your level of experience with Unix systems?

> Intermediate. I use macOS command line and I've used Linux VMs for school.

What, if any, experience do you have maintaining systems for other people?

> For my research assistant work, I was benchmarking the tool we were
> working on. I had to check `top`, `free`, `df`, and other coreutils,
> run `docker stats` and save the output, help write some scripts to
> run our benchmarks, etc.

How much programming experience do you have?
Please enter approximate KLOC and programming languages with rough skill level (novice, experienced, expert).

> at least 5KLOC, probably closer to 10K
> C - experienced - not sure what counts as expert. I haven't tried programming
> with arena style memory management. I might try that in the future.
> C++ - novice - considering I only really know the C-style C++ (or std C++98)
> and not many modern C++ features
> Rust - novice
> Python - somewhat experienced

Please read the paragraph entitled "Plagiarism, Cheating and other ways to get an F" on the course website.
Please enter below a statement that you have read and understood its meaning and implications.

> I have read the text under that section and the blog post and the
> permitted use of AI technologies and have understood the meaning
> and implication. I won't forget to share a link to any AI prompting
> that I do.

---

Meetup Requirement:


Weekly Presentation date:
Partner - Andrew Schomber


Little Snitch - monitors laptop and checks whenever some software wants to make an outbound connection.

We need a dualstack system IPv4 and v6 support.

Host *amazonaws.com
    User root
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

[jschauma aliases](https://raw.githubusercontent.com/jschauma/cloud-functions/refs/heads/main/awsfuncs)

https://www.netmeister.org/blog/ec2-ipv6.html

ping runs ICMP, which run on raw sockets
