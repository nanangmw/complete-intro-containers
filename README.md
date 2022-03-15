
# Crafting Containers By Hand

### What Are Containers

The core of what containers are is just a few features of the Linux kernel duct-taped together, it's just using a few features of Linux together to achieve isolation.

containers give us many of the security and resource-management features of VMs but without the cost of having to run a whole other operating system. It instead usings chroot, namespace, and cgroup to separate a group of processes from each other.

### chroot

chroot often called as "cha-root" and "change root", it's a Linux command that allows you to set the root directory of a new process. In our container use case, we just set the root directory to be where-ever the new container's new root directory should be. And now the new container group of processes can't see anything outside of it, eliminating our security problem because the new process has no visibility outside of its new root.

### Namespaces

While chroot is a pretty straightforward, namespaces and cgroups are a bit more nebulous to understand but no less important. Both of these next two features are for security and resource management. Namespaces allow you to hide processes from other processes. If we give each chroot'd environment different sets of namespaces, now users can't see each others' processes (they even get different process PIDs, or process IDs, so they can't guess what the others have) and you can't steal or hijack what you can't see!

`unshare`  creates a new isolated namespace from its parent and all other future tenants. Run this:

```
exit # from our chroot'd environment if you're still running it, if not skip this

# install debootstrap
apt-get update -y
apt-get install debootstrap -y
debootstrap --variant=minbase bionic /better-root

# head into the new namespace'd, chroot'd environment
unshare --mount --uts --ipc --net --pid --fork --user --map-root-user chroot /better-root bash # this also chroot's for us
mount -t proc none /proc # process namespace
mount -t sysfs none /sys # filesystem
mount -t tmpfs none /tmp # filesystem

```

### cgroups

Every isolated environment has access to all physical resources of the server. There's no isolation of physical components from these environments.

Enter the hero of this story: cgroups, or control groups. Google saw this same problem when building their own infrastructure and wanted to protect runaway processes from taking down entire servers and made this idea of cgroups so you can say "this isolated environment only gets so much CPU, so much memory, etc. and once it's out of those it's out-of-luck, it won't get any more."

This is a bit more difficult to accomplish but let's go ahead and give it a shot.

```
# outside of unshare'd environment get the tools we'll need here
apt-get install -y cgroup-tools htop

# create new cgroups
cgcreate -g cpu,memory,blkio,devices,freezer:/sandbox

# add our unshare'd env to our cgroup
ps aux # grab the bash PID that's right after the unshare one
cgclassify -g cpu,memory,blkio,devices,freezer:sandbox <PID>

# list tasks associated to the sandbox cpu group, we should see the above PID
cat /sys/fs/cgroup/cpu/sandbox/tasks

# show the cpu share of the sandbox cpu group, this is the number that determines priority between competing resources, higher is is higher priority
cat /sys/fs/cgroup/cpu/sandbox/cpu.shares

# kill all of sandbox's processes if you need it
# kill -9 $(cat /sys/fs/cgroup/cpu/sandbox/tasks)

# Limit usage at 5% for a multi core system
cgset -r cpu.cfs_period_us=100000 -r cpu.cfs_quota_us=$[ 5000 * $(getconf _NPROCESSORS_ONLN) ] sandbox

# Set a limit of 80M
cgset -r memory.limit_in_bytes=80M sandbox
# Get memory stats used by the cgroup
cgget -r memory.stat sandbox

# in terminal session #2, outside of the unshare'd env
htop # will allow us to see resources being used with a nice visualizer

# in terminal session #1, inside unshared'd env
yes > /dev/null # this will instantly consume one core's worth of CPU power
# notice it's only taking 5% of the CPU, like we set
# if you want, run the docker exec from above to get a third session to see the above command take 100% of the available resources
# CTRL+C stops the above any time

```

And now we can call this a container.

# Docker

Docker is a commandline tool that made creating, updating packaging, distributing, and running containers significantly easier which in turns allowed them become very popular with not just system administraters but the programming populace at large. At its heart, it's a command line to achieve what we were doing with cgroups, namespaces, and chroot but in a much more convenient way.

### Docker Desktop

Go ahead and install  [Docker](https://www.docker.com/products/docker-desktop)  Desktop right now. It will work for both Mac and Windows.

If you're on Mac, you'll see a cute little whale icon in your status bar. Feel free to poke around and see what it has. It will also take the liberty of installing the  `docker`  commandline tool so we can start doing all the fun things with Docker.

### Docker Hub

Docker Hub is a public registry of pre-made containers, you can visit it  [here](https://hub.docker.com/search?q=&type=image). Instead of having to handcraft everything yourself, you can start out with a base container from Docker Hub and start from there.

### Docker Images without Docker

These pre-made containers are called images. They basically dump out the state of the container, package that up, and store it so you can use it later. So let's go nab one of these image and run it! We're going to do it first without Docker to show you that you actually already knows what's going on. Let's grab the latest Node.js container that runs Ubuntu.

```
# start docker contanier with docker running in it connected to host docker daemon
docker run -ti -v /var/run/docker.sock:/var/run/docker.sock --privileged --rm --name docker-host docker:18.06.1-ce

# run stock alpine container
docker run --rm -dit --name my-alpine alpine:3.10 sh

# export running container's file system
docker export -o dockercontainer.tar my-alpine

# make container-root directory, export contents of container into it
mkdir container-root
tar xf dockercontainer.tar -C container-root/

# make a contained user, mount in name spaces
unshare --mount --uts --ipc --net --pid --fork --user --map-root-user chroot $PWD/container-root ash # this also does chroot for us
mount -t proc none /proc
mount -t sysfs none /sys
mount -t tmpfs none /tmp

# here's where you'd do all the cgroup rules making with the settings you wanted to

```

### Docker Images with Docker

Run this command:

> docker run --interactive --tty alpine:3.10 # or, to be shorter: docker run -it alpine:3.10

then try :

> docker run alpine:3.10 ls

The  `ls`  part at the end is what you pass into the container to be run. As you can see here, it executes the command, outputs the results, and shuts down the container.

So now what if we want to detach the container running from the foreground :

> docker run --detach -it ubuntu:bionic # or, to be shorter: docker run -dit ubuntu:bionic

So it prints a long hash out and then nothing. To get ahold of it :

> docker ps

This will print out all the running containers that Docker is managing for you. You should see your container there. So copy the ID or the name :

> docker attach # e.g.  `docker attach 20919c49d6e5`  would attach to that container

This allows you to attach a shell to a running container and mess around with it. Useful if you need to inspect something or see running logs. Run  `docker run -dit ubuntu:bionic`  one more time. Let's kill this container without attaching to it. Run  `docker ps`, get the IDs or names of the containers you want to kill and say:

> docker kill # e.g.  `docker kill fae0f0974d3d 803e1721dad3 20919c49d6e5`  would kill those three containers

A fun way to kill all running containers would be

> docker kill $(docker ps -q)

The  `$()`  portion of that will evaluate whatever is inside of that first and plug its output into the second command. In this case,  `docker ps -q`  returns all the IDs and nothing else. These are then passed to  `docker kill`  which will kill all those IDs.

### --name and --rm

you can refer to these by a name you set by running :

```
docker run -dit --name my-ubuntu ubuntu:bionic
docker kill my-ubuntu

```

But now if you tried it again, it'd say that  `my-ubuntu`  exists. If you run  `docker ps --all`  you'll see that the container exists even if it's been stopped. That's because Docker keeps this metadata around until you tell it to stop doing that. You can run  `docker rm my-ubuntu`  which will free up that name or you can run  `docker container prune`  to free up all existing stopped containers. In the future you can just do :

```
docker run --rm -dit --name my-ubuntu ubuntu:bionic
docker kill my-ubuntu

```

### Node.js on Docker

he default Ubuntu container doesn't have Node.js installed. Let's use a different container!

> docker run -it node:12-stretch

Notice this drops us into the Node.js REPL which may or may not be what you want. What if we wanted to be dropped into bash of that container?

> docker run -it node:12-stretch bash

### Tags

If you run  `docker run -it node`  the tag implicitly is using the  `latest`  tag. When you say  `docker run -it node`, it's the same as saying  `docker run -it node:latest`. The  `:latest`  is the tag. This allows you to run different versions of the same container, just like you can install React version 15 or React version 16: some times you don't want the latest.

### Docker CLI

#### pull / push

`pull`  allows you to pre-fetch container to run.

```
docker pull jturpin/hollywood
docker run -it jturpin/hollywood hollywood # notice it's already loaded and cached here; it doesn't redownload it

```

That will pull the hollywood container from the user jturpin's user account. The second line will execute this fun container which is just meant to look a hacker's screen in a movie (it doesn't really do anything than look cool.)

`push`  allows you to push containers to whatever registry you're connected to (probably normally Docker Hub or something like Azure Container Registry).

#### inspect

> docker inspect node

This will dump out a lot of info about the container. Helpful when figuring out what's going on with a container

#### pause / unpause

these pauses or unpause all the processes in a container.

```
docker run -dit jturpin/hollywood hollywood
docker ps # see container running
docker pause <ID or name>
docker ps # see container paused
docker unpause <ID or name>
docker ps # see container running again
docker kill <ID or name> # see container is gone

```

#### exec

This allows you to execute a command against a running container. This is different from  `docker run`  because  `docker run`  will start a new container whereas  `docker exec runs`  the command in an already-running container.

```
docker run -dit jturpin/hollywood hollywood
docker ps # grab the name or ID
docker exec <ID or name> ps aux # see it output all the running processes of the container

```

#### history

this allow you to see how this Docker image's layer composition has changed over time and how recently.

> docker history node

#### info

Dumps a bunch of info about the host system.

> docker info

#### top

Allows you to see processes running on a container

```
docker run mongo
docker top <ID outputted by previous command> # you should see MongoDB running

```

#### rm / rmi

If you run  `docker ps --all`  it'll show all containers you've stopped running in addition to the runs you're running. If you want to remove something from this list, you can do  `docker rm <id or name>`.

If you want to remove an image from your computer you can run  `docker rmi mongo`  and it'll delete the image from your computer. This isn't a big deal since you can always reload it again

#### logs

To see the output of one of your running containers.

```
docker run -d mongo
docker logs <id from previous command> # see all the logs

```

#### search

This will allow you to take a look.

```
docker search python # see all the various flavors of Python containers you can run
docker search node # see all the various flavors of Node.js containers you can run

```

# The Dockerfile

A big key with Docker container is that they're supposed to be disposable. You should be able to create them and throw them away as many times as necessary. In other words: adopt a mindset of making everything short-lived. There are other, better tools for long-running, custom containers.

```
FROM node:12-stretch

CMD ["node", "-e", "console.log(\"hi lol\")"]

```

The first thing on each line (`FROM`  and  `CMD`  in this case) are called instructions.

### A Note on EXPOSE

There is an instruction called  `EXPOSE <port number>`  that its intended use is to expose ports from within the container to the host machine. However if we don't do the  `-p 3000:3000`  it still isn't exposed so in reality this instruction doesn't do much. You don't need  `EXPOSE`.

# Features in Docker

### Bind Mounts

Bind mounts allow you to mount files from your host computer into your container. This allows you to use the containers a much more flexible way than previously possible: you don't have to know what files the container will have when you build it and it allows you to determine those files when you run it.

```
# from the root directory of your CRA app
docker run --mount type=bind,source="$(pwd)"/build,target=/usr/share/nginx/html -p 8080:80 nginx

```

This is how you do bind mounts. It's a bit verbose but necessary.

-   We use the  `--mount`  flag to identify we're going to be mounting something in from the host.
-   As far as I know the only two types are  `bind`  and  `volume`. Here we're using bind because we to mount in some piece of already existing data from the host.
-   In the source, we identify what part of the host we want to make readable-and-writable to the container. It has to be an absolute path (e.g we can't say  `"./build"`) which is why use the  `"$(pwd)"`  to get the present working directory to make it an absolute path.
-   The target is where we want those files to be mounted in the container. Here we're putting it in the spot that NGINX is expecting.
-   As a side note, you can mount as many mounts as you care to, and you mix bind and volume mounts. NGINX has a default config that we're using but if we used another bind mount to mount an NGINX config to  `/etc/nginx/nginx.conf`  it would use that instead.

### Volumes

Volumes works so your containers can maintain state between runs. Volumes can not only be shared by the same container-type between runs but also between different containers.

### named pipes, tmpfs, and wrap up

`tmpfs`

> `tmpfs`  imitates a file system but actually keeps everything in memory. This is useful for mounting in secrets like database keys.

`npipe`

> `npipe`  useful for mounting third party tools for Windows containers.
