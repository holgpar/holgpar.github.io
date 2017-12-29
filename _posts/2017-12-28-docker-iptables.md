---
title: Docker with iptables from the inside
date: 2017-12-28
categories: docker kubernetes networking container linux
excerpt_separator: <!--more-->
---

# Docker and Kubernetes with iptables from the inside

I am going to describe how to configure a firewall via iptables
for a single container or a bunch of containers sharing the same network interface (think "pod").

<!--more-->

In the first part, we will focus on plain Docker and then we will repeat the exercise with Kubernetes.

## But why?

So, why would one want to do that?
In a bigger organization, you might have the need to run processes which are only partially trusted in a central infrastructure.
For example you might be doing release builds of software products.
Most likely, your "release grade" builds have to adhere to a set of compliance regulations.

Let's say those builds will not be allowed to access arbitrary repositories to pull their dependencies from,
but only certain repositories under your control which are also of "release grade".

The underlying compliance requirement might be that your builds need to be reproducible at a later point in time,
or simply that you want to control what code ends up in your products.

So how could a technical implementation of this requirement look like?
Easy, just compile a white list of all allowed repositories and download sites and use this to setup outbound firewall rules for your build server. Done!

Well, yes. But this is not very flexible. Imagine there are also other types of builds,
call them "dev" builds, for which only relaxed requirements are enforced.
For those builds, we would then need other build servers with different firewall configurations.
That would create a partitioning of our pool of build servers which is not desirable from the point of view of resource utilization.
The aim is to be able to run every build on every sever.
The firewall rules should be defined dynamically, depending what the quality grade of the build ("dev" or "release") is.

That is the scenario for which I started to think about how firewall rules can be defined for specific build processes,
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

Okay. Obviously we are root. But since we are running in a container, we might miss a couple of Linux capabilities.
The capability we need is called 'NET_ADMIN':

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
We could work around that by doing user switches, but it would be much nicer if we would not have to mess with the container
executing the build process.

The idea is to use to containers: One to define the iptables rules, and one to execute the payload build process.
That way, we achieve a nice separation of concerns: One container per purpose.
The respective images can than more easily be prepare by different people.

In order to realize this, we kind of backport the pod concept from Kubernetes to Docker.
We create a two container sharing the same network interface the same way Kubernetes does:


```bash
# start container with NET_ADMIN capability and set up firewall
$ docker run --name=iptables -d --cap-add NET_ADMIN ubuntu tail -f /dev/null
$ docker exec iptables iptables -A OUTPUT -d 10.96.48.117 -j DROP

# start the payload container in the same network
$ docker run -it --name=payload --net=container:iptables ubuntu /do_something.sh
```

You can exec into the payload container and see for yourself.
Your will be subjected to firewall rules without being able to change them.
Once the payload container finished whatever it had to do, you can stop the iptables container
and Docker will clean up the virtual network interface as well as the iptable rules for the network namespace.

## The Kubernetes way

What we did in the Docker case draws its inspiration form Kubernetes.
So it comes with no surprise that doing the same thing with Kubernetes is straightforward.
Here is pod manifest:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: iptables-test
spec:
  initContainers:
    - name: iptables
      image: docker.wdf.sap.corp:51062/tests/iptables
      command: [ "bash", "-c", "iptables -A OUTPUT -d 10.96.48.117 -j DROP" ]
      securityContext:
       capabilities:
         add:
           - NET_ADMIN
  containers:
    - name: payload
      image: ubuntu
      command: ["tail", "-f", "/dev/null"]
```

We make use of the Kubernetes concept of init containers.
That way, we can ensure that the setup of the firewall is completed before the payload container starts.

[Kubernetes network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
are an alternative to the above technique that should be considered.
If you can use them, you should use them, because you will benefit from the declarative approach of Kubernetes.

However, if network policies work depends on the your network plugin.
Also, using the above technique you have the full power and flexibility of iptables at your command.
(No pun intended).

### Further reading

Behind the scene the above examples rely on network namespaces.
To learn more about this topic (it's worth it) start with [blog post of Scott](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)
