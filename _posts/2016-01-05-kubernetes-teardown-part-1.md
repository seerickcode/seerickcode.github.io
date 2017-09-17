---
layout: post
title: Kubernetes - Teardown - Part 1
modified:
categories: 
description:
tags: []
image:
  feature:
  credit:
  creditlink:
comments: True
share: True
---

https://en.wikipedia.org/wiki/Ship's_wheel#/media/File:Steering_gear_18th_century-numbered.svg

# Teardown of Kubernetes - Part 1

So I need to do an trial of using Google Container Engine for a client to host some docker infrastructure
we already have in place.  Well, Google is pushing Kubernetes, and there is pleny of documentation, etc.
to jump right in and start playing with some demo containers, etc.

I don't want to use up the free credits I have for the investigation just learning Kubernetes cluster 
management, I want to use that for testing/debugging the actual application stack.  With that in mind
I start down path of setting up Kubernetes on my local workstation, all inside docker containers.

ie.. This http://kubernetes.io/v1.1/docs/getting-started-guides/docker.html

Unfortunatly, there is just a lot of cut-and-paste to get started without much explanation.  Sort of a 
Wizard of Oz, 'Don't look behind the curtain, Just accept that we are awsome, and give us a docker
image to run, and we will get right on it..'...

Well, at Step Two - the docker container it starts decides to crap all over my /dev and now I can't open
new terminal sessions (turns out to be a bug where it changes the permissions on /dev/pts/something).

Now I need to pull back the curtain if only to give a piece of my mind to whatever is back there for letting
this happen.  

## Behind the curtain.
All these docker images are coming from Google's own docker hub.  They are build from image sources you can
check out in the (Kubernetes github repo)[https://github.com/kubernetes/kubernetes].  They are under cluster/images

### Step One - etcd
The first thing they want us to start is the etcd server docker image.  Actually nothing very exciting here.
It is just build a Busybox container with the precompiled etc binaries stuffed in.  It runs with --net=host so 
its networking space is basically your host (ifconfig in container looks like ifconfig on parent)

It binds up etcd on localhost:4001

That is about all that needs to be said for Step One - The EtcD container.  How EtcD is used we will find out 
later - I hope :)

#### Side note - (nsenter)[https://blog.docker.com/tag/nsenter/]
Handy utility to spawn a process inside a running docker container.  Used this a bit vs docker exec to get into
the containers and check them out.  Also see (this)[http://stackoverflow.com/questions/27873312/docker-exec-versus-nsenter-any-gotchas]

### Step Two - Run the Master

Ok, so this is the one that I needed to grok a bit more since it is a key player, and is sort of glossed
over in the main documentation as "This actually runs the kubelet, which in turn runs a pod that contains the other master components."

Its docker image setup lives under cluster/images/hyperkube in the Repo.  The Makefile and the Dockerfile are 
what we need to focus on.  

Also - I am working of the current release 1.1.3 tag, not master branch.  There is a lot of new stuff in master.

Makefile - Pretty straight forward - pulls down a prebuilt 'hyperkube' program and does some sed handywork on 
some json config files.

Dockerfile - Fairly basic - based off debian:jessie, copies nsenter binary in the container up to /, adds
in the downloaded hyperkube binary in /, pushes in the json config files under /etc/kubernetes, and adds in
some script filefs to do a safe_format_and_mount.  Not sure what the last one is needed for yet, but looks
scary espesccially with me binding a bunch of my host directories into the image :)

Now we hit the actual docker run command.  
<code>
docker run \
    --volume=/:/rootfs:ro \
    --volume=/sys:/sys:ro \
    --volume=/dev:/dev \
    --volume=/var/lib/docker/:/var/lib/docker:ro \
    --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
    --volume=/var/run:/var/run:rw \
    --net=host \
    --pid=host \
    --privileged=true \
    -d \
    gcr.io/google_containers/hyperkube:v1.0.1 \
    /hyperkube kubelet --containerized --hostname-override="127.0.0.1" --address="0.0.0.0" --api-servers=http://localhost:8080 --config=/etc/kubernetes/manifests
</code>

Quite a command..

The --net, --pid, --privileged basically give this container full run of your host system.  This makes sense
for this type of container as it seems it will be causing the launch and management of other docker systems.
I thought there was a Docker API for all this that containers could use, but whatever, I guess they didn't like
that path, or it was not ready for primetime.

It runs the hyperkube command with the 
`kubelet --containerized --hostname-override="127.0.0.1" --address="0.0.0.0" --api-servers=http://localhost:8080
--config=/etc/kubernetes/manifests`
parameters.

#### Hyperkube..
I researched what is going on here.  Looks like Hyperkube is a wrapper exec of various other 'programs'.
It took a bunch of searching but godoc.org had some docs.

<quote>
Package hyperkube is a framework for kubernetes server components. It allows us to combine all of the kubernetes server components into a single binary where the user selects which components to run in any individual process.

Currently, only one server component can be run at once. As such there is no need to harmonize flags or identify logs across the various servers. In the future we will support launching and running many servers -- either by managing processes or running in-proc.

This package is inspired by https://github.com/spf13/cobra. However, as the eventual goal is to run *multiple* servers from one call, a new package was needed.
</quote>

If you open up the json files, you will see some of the things that hypercube is going to be called as.  This is 
sort of like Busybox where one executable contains all the parts of the individual programs.  I am guessing there
is some Go magic to bundle this all together nice, and that is why they decided to do it.  Seems fairly elegant.

So, 'kubelet' is the first one we need to focus on.  It looks like the json files will lead to others being
spawned and we will tackle those in time.

Seems there are a lot of people asking the same questions - 'What is Kubelet?'.  Since there is a lot of reading material
available on that I will not go into too much detail here.

The deprecated Kerbernetis.io/v1.0 (docs)[http://kubernetes.io/v1.0/docs/admin/kubelet.html] show it as 

<quote>
The kubelet is the primary "node agent" that runs on each node. The kubelet works in terms of a PodSpec. A PodSpec is a YAML or JSON object that describes a pod. The kubelet takes a set of PodSpecs that are provided through various mechanisms (primarily through the apiserver) and ensures that the containers described in those PodSpecs are running and healthy.

Other than from an PodSpec from the apiserver, there are three ways that a container manifest can be provided to the Kubelet.

File: Path passed as a flag on the command line. This file is rechecked every 20 seconds (configurable with a flag).

HTTP endpoint: HTTP endpoint passed as a parameter on the command line. This endpoint is checked every 20 seconds (also configurable with a flag).

HTTP server: The kubelet can also listen for HTTP and respond to a simple API (underspec'd currently) to submit a new manifest.

</quote>

The options:
- --containerized - 'Experimental support for running kubelet in a container'  - Great..
- --api-servers - List of api-servers to connect to for getting pod config details
- --config - Path to the json files that will be read instead of just the API Servers.  This is the master.json that gets pushed into the container image.

Ok, so this is pretty cut and dry, the container starts up and runs a pod manager, and the first pod setup it 
builds is specified by the json file master.json

Sure enough, the master.json looks like a normal Kubernetes pod syntax, in json instead of yaml.

It is going to spin up the following new docker containers:
- controller-manager
- apiserver
- scheduler

Each using the same hyperkube image, but with different first commands.  Effectively boot strapping this whole mess.

What is the mess ?

#### Controller-Manager

Official doc is (here)[https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/admin/kube-controller-manager.md]

This seems to be a daemon that will just wrangle all the other 'controllers' (replication, endpoints, namespace, servivceaccounts, future) to keep a certain state enforced by reading the status from the API server.

Options getting passed in:
- --master=127.0.0.1:8080 - This will be what the 'apiserer' will bind as later when its docker instance kicks off
- --v=2 - Not sure - It is undocumented.  Could be an api version or perhaps a verbosity

#### API Server

Straight forward enough - central collection for all the pushed in 'configuration' for Kubernetes, and probably
also other collected data.  Based on its configuration, it looks like it uses etcd to store its key/value pairs.

The (official docs)[http://kubernetes.io/v1.1/docs/admin/kube-apiserver.html] on this show a metric butt load of options you can review at your leisure
Options getting passed in:
- --portal-net=10.0.0.1/24 - Undocumented - Great.. It looks like this is simply a default subnet for instances.
- --address=127.0.0.1 - What it will bind to.
- --etcd-servers=http://127.0.0.1:4001 - The address:port of the etcd server we started up
- --cluster-name=kubernetes - Another default - I would think so that multiple api servers can get their co-ordinated act together. 
- --v=2 - Not sure - It is undocumented.  Could be an api version or perhaps a verbosity

#### Scheduler

From the (official docs)[http://kubernetes.io/v1.1/docs/admin/kube-scheduler.html] :
<quote>
The Kubernetes scheduler is a policy-rich, topology-aware, workload-specific function that significantly impacts availability, performance, and capacity. The scheduler needs to take into account individual and collective resource requirements, quality of service requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference, deadlines, and so on. Workload-specific requirements will be exposed through the API as necessary.
</quote>

Egads - Looks like someone had that generated by a buzzword generator..

This (Stack Overflow Post)[http://stackoverflow.com/questions/28857993/how-does-kubernetes-scheduler-work/28874577] explains it a bit better.

For now, what the scheduler does is when a pod needs to be activated (a 'pod' is just a bunch of docker containers
that need to stay together) it looks at the available places it can be activated on and using some policies, 
decides where it is going to turn it all on.  Remember, there are normally going to be a bunch of actual machines
either bare metal, or full virtual machines that all our docker images are going to be activated on.

### Step 3 - Run the service proxy

Well, step 2 was quite an undertaking, but ew are now down to the last part, the Service Proxy.

This is another hyperkube image, and it runs the 'proxy' command.
The (official docs)[https://github.com/kubernetes/kubernetes/blob/master/docs/admin/kube-proxy.md] show quite a few
options.  The ones used in the demo setup are :

- --master=http://127.0.0.1:8080 - The uri to the API Server
- --v=2 - Not sure - It is undocumented.  Guessing it is API version now..

The service proxy, from what see/read is where all the 'role=loadbalancer' pod configuration ends up
and from here it calls certain provider specific functions to setup proxy services and/or performs some
simple routing itself.  On bare metal, it looks like you might need other items to fill the actual 
routing handling - ie, the HaProxy service-loadbalancer (here)[https://github.com/kubernetes/contrib/tree/master/service-loadbalancer]

We might dig into this item a bit more in a future part 2 where we actually start using this stuff.

### Wrap up

So we know have a better understanding of what exactly is going on behind the 'wheel' to setup this
kubernetes system on your local box.  The 'hello world' examples that they get you to run now might make
a bit more scense when you have the bigger picture in your mind.  It certainly helps for me, I hope it helps
for you.

I will update this post with the Part 2 when/if I get around to it.  I am off to play with Kubernetes now :)

### Oh yeah !
I forgot to mention about the --volume=/dev:/dev in the Step Two docker stand up.
I have removed it from mine.  I think I can see some reasons why the 'kkubelet' might need 
full access to my /dev directory, but until I hit that requirement I am going to keep it 
the heck out of my /dev since it buggers it up.

If you find yourself unable to open new terminal sessions after starting the kubelet instance, do a 
<code>
sudo chmod 666 /dev/pts/ptmx
</code>

... to get you back into action (assuming you still have one terminal going that is :) )











 

