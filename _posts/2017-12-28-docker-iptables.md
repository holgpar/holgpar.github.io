---
title: Docker with iptables from the inside
date: 2017-12-28
---

# Docker with iptables from the inside

In this post I am going to describe how to configure a firewall via iptables
for a single container or a bunch of containers sharing the same network interface (think "pod").

In the first part of the article, we will focus on plain Docker and then we will repeat the exercise with Kubernetes.

## But why?

So, why would one want to do that? By default, your containers will not be accessible form the outside
until you explicitly expose ports.
For most scenarios, that is already enough as a firewall shielding your containers form unwanted ingress traffic.

My main motivation to is to restrict egress traffic in a container specific way.
That can very useful if your are doing release builds of software products:
Let's says that builds of the highest quality level have to adhere to a set of compliance regulations.

Most likely those builds will not be allowed to access arbitrary repositories to pull there dependencies from,
but only certain repositories under your control which are also of "release" grade.

The underlying compliance requirements might be that your builds need to be reproducible at a later point in time,
or simply that you want to control what code ends up in your products.

So how could a technical implementation of this requirement look like?
Easy, just compile a white list of all allowed repositories and download sites and use this to setup outbound firewall rules for your build server. Done!

Well, yes. But now, image there are also other types of builds lets call them "dev" builds, for which only relaxed requirements are enforced.
For those builds, we would then need other build servers with different firewall configurations.
That would create a partioning of our pool of build servers which is not desirable from the point of view of resource utilization.
The aim is to be able to run every build on every sever.
The firewall rules should be defined dynamically, depending what the quality level of the build ("dev" or "release") is.

In that situation I started to think about how firewall rules can be defined for specific build processes,
each build process fortunately running inside a container.

## The plain Docker way

For each container, Docker creates an own virtual network interface connected to the Docker bridge.
So in principle, we should be able to create iptable rules for that interface. Let's see, how we get there.


First, lets create a simple image having the iptables command line tool on board.

````Dockerfile
FROM ubuntu

RUN apt-get update && apt-get install -y iptables
````

easy. We build it and give it the name `iptables`:

````
docker build -t iptables .
````

Let's give it a try:

```
$ docker run --rm iptables  iptables -L
iptables v1.6.0: can't initialize iptables table `filter': Permission denied (you must be root)
Perhaps iptables or your kernel needs to be upgraded.
```

Okay. Obviously we are root. But since we are running in a container, we might miss a couple of linux capabilities.
The capabilty we need is called 'NET_ADMIN':

```
$ docker run --rm --cap-add NET_ADMIN iptables  iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
```

Much better. Compare this with the output of

```
# iptables -L
```

directly on your host. You will notice that your container cannot see any of those rules.
That is because it has its own network interface and lives in an own network namespace.

So now we can in fact define iptables rules which only effect our container:

```
$ docker run --rm iptables iptables -A OUTPUT -d 10.96.48.117 -j DROP
```

Try this in an interactive shell and afterwards try to access the blocked ip.
You will see it fails while on host level you can access it without any problems.


But wait, our original purpose was to run a process which is not fully trusted in an restricted network environment.
This process could easily get rid of the iptable rule, because the container still has the capability `NET_ADMIN`.
We could work around that by doing user switches, but would be much nicer if we would not have to mess with the container
executing the build process.

The idea is to use to containers: One to define the iptables rules, and one to execute the payload build process.
That way, we achieve a nice separation of concerns: One container per purpose.
The respective images can than more easily be prepare by different people.

In order to realize this, we kind of backport the pod concept from Kubernetes to Docker.
We create a two container sharing the same network interface the same way Kubernetes does:


```
docker run --name=iptables -d --cap-add NET_ADMIN ubuntu tail -f /dev/null
docker exec
```
