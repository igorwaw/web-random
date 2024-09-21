Title: Switching from Docker to Podman
Date: 2024-09-21 16:00
Status: published
Category: random
Tags: linux
Slug: from-docker-to-podman

Many distributions have switched from Docker to other container engines. There were different reasons for it - licensing or design choices - but for years I ignored the trend. I thought: container engines are supposedly 99% compatible with Docker, after all they all claim to support OCI standard, but somehow I alway run into this 1% of incompatibility. So I just always installed Docker to avoid the issues.

Few months ago the principal engineer at my workplace, who's a big fan of Podman, encouraged me to give it another try - to spend a few hours learning about the differences before I decide that something doesn't work with Podman. I quickly discovered things that are obvious to any Podman user: while the command line tool is compatible and you can do "alias docker=podman", there are some deeper differences. Once you learn them, you can do all the same things as with Docker, plus some more.

Short version: whenever there's a choice between convenience and security, Docker defaults to convenience and Podman to security.

## Container registries

Let's start with the simplest one. Check any tutorial on Docker and it will start with "docker run -it ubuntu" or something similar. Such command will probably fail with Podman. Docker tries to run a locally available image, if it doesn't find one, it downloads it from docker.io registry. You can point it to another registry by giving full path, but that's the default one.

Anyone can upload their software to docker.io registry. Obviously it has some good sides - a huge range of images to choose from and a place to store your own stuff. But that also means you can accidentally download some low quality code. Or worse, malicious code that steals your data. Many companies forbid the use of public container registries.

Podman has no default container registry (unless your distro already changed the default configuration). You can configure one (or multiple) in /etc/containers/registries.conf, it might be your company's internal server, might be docker.io or another public one eg. quay.io. Or you can provide full path: "podman run -it docker.io/ubuntu". Or you can configure shortnames only for specific images - for example Ubuntu does that with their Podman package, meaning that "podman run" will work with ubuntu, rhel, alpine, python and many other well-known images, but now with some random stuff found on the net. All of these don't prevent the problem of downloading the bad stuff, but it makes it a bit harder to do by accident. If you don't agree, it's easy to change the default to the same as Docker's.

## Networks and pods

Let's now move from Docker to Docker Compose tutorial. It usually involves running several containers, let's say: fronted, backend and database. Each one can connect to others using their names. That won't work with Podman - unless you inform Podman that this is your intention. For example, by placing then in the same pod.

The concept of pods comes from Kubernetes. Pods contain several containers grouped together - and they obviously are supposed to communicate. They can also be started/stopped together. But they really show their strength when you move to Kubernetes - see few points below.

There are also other ways to connect the containers, you're not forced to use pods. You can use the host's hostname if the ports are exposed. Or configure a private network, they have name resolution. Or configure a service mesh or a discovery service. Or, if you insist, you can enable dnsname plugin on the default network (the one simply called podman). Podman is just missing one shortcut option that's present in Docker because, again, it makes it harder to do insecure things by accident.

## Rootless

The biggest difference between Docker and Podman. By default, Docker runs as root. Even if you run docker command as a user, it talks with the daemon that runs as root - and so the containers are started as root. Podman, on the other hand, really runs with the privilges of the user who invoked it. This has some serious implications:

- The good: Podman is much more secure by design. If Docker fails to properly isolate the container (and there were such vulnerabilities in the past), the container can get full root rights to the host system and other containers. With Podman, the worst case scenario is getting access to one user account.
- The bad: some containers need more than 1 user id. Podman can provide a full set of 2^16 ids using subids. No big deal, but: you have to have a proper configuration in /etc/subuid and /etc/subgid - with any modern distro and the standard way of creating accounts, you usually do, if you do something uncommon you might have to manually edit the file. And if your containers bind mount directories from the host filesystem, you may run into permission problems. Again, quite easily fixable with "podman unshare chown" or ":U" option for the volume, but more than one person ran into this and decided that Podman doesn't work.
- The ugly: some, but very few containers, really need extra privileges. If that is the case, you know what you're doing and you're accepting security implications, you can simply run podman as root. You're opting out of extra isolation, but at least it's for the selected few containers, not all of them. Even better, you can fine-tune the security options by adding and removing capabilities and seccomp filters, it's not only the choice between full root access and nothing.

## Daemonless

Podman doesn't run a daemon by default. When you run a podman command, it will do what you requested (eg. run a container, delete an image, query for information) and simply exit. Again, security - no huge binaries running all the time, no exposed ports. You can choose to run a daemon if it's needed for better Docker compatibility eg. with a 3rd party tool.

## Systemd for starting containers

Docker doesn't play well with Systemd. It's the design choice - both tools try to config cgroups and namespaces, restart services etc. Docker developers plainly refuse to cooperate, they want complete control. On the other hand, Podman authors decided it's not the job of the container engine to autostart services. It's either done with an orchestrator such as Kubernetes, or, for running on a single machine, you can just use the same software that starts other services, which is systemd.

Just one command, "podman generate systemd" will create the config file for autostarting your service. And you can even run it as user (yes, systemd supports that). Instead of autostarting at boot, you can also attach it to a socket and only activate it to handle incoming connection (if you're as old as me, you remeber inetd doing the same thing).

Fine print: "generate systemd" is now deprecated and Podmans wants you to use quadlets. Quadlets are more powerful, but also more complex, which spoils my narrative. But the old command still works.

## Systemd inside the containers

The most common way to use containers today is to isolate a single application. One container, one process, if it exits, the whole container exits (and possibly gets restarted). But there is another approach - treat container as something like a VM, running multiple daemons. Anything in between is also possible.

Personally, I'm in a single process camp. It's been several years since I ran multiple processes (and that was because I didn't know how to handle multi-container setup). But I think it's better to have a choice. If you need to run systemd inside the container, Podman will detect it and automatically set the environment properly.

## Kubernetes

Kubernetes used to use Docker, but switched to other container engines (CRI-O by default). Yet it's Podman that works better with it.

The idea is: you use Podman to develop and test software and then you can easily run it in production on Kubernetes. Podman, unlike Docker, has a concept of pods. If you're ready to move your software from a single machine to Kubernetes cluster, you can use "podman generate kube" to create the YAML file in the Kubernetes format. Since the format is not easy to create by hand, it saves a lot of time.

But there's more. What if you have some software already running on Kubernetes, but want to test it on a single machine? There's "podman play kube" which reads Kubernetes YAML. Of course it only accepts a subset of options as Podman can't do scaling and multi-node deployment.
