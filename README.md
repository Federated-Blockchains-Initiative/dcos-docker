## DC/OS Docker

Run DC/OS (including systemd) in Docker containers, one per node, using Docker-in-Docker for DC/OS services/jobs.

DC/OS Docker is designed to optimize developer cycle time. For a more production-like local experience, see [DC/OS Vagrant](https://github.com/dcos/dcos-vagrant) which runs each node in its own VM.

## Caveats

- Because of Docker-in-Docker, DC/OS services (like Jenkins) that themselves use Docker-in-Docker may not work correctly.
- Because containerization does not affect resource detection tools, each DC/OS node will think it can allocate all of the host's resources, leading to over-subscription without protection. You are still bound by the disk and memory constraints of the host, even if DC/OS thinks you have N (number of agent nodes) times more. Running Docker in a VM can protect your host from this and allows you to designate how much disk/memory/cpu DC/OS gets in total. Running Docker directly on a Linux host gives DC/OS more resources to play with but may also freeze your machine if you run too many DC/OS services/jobs.

## Requirements

DC/OS Docker can be run on macOS or Linux, or on Vagrant with VirtualBox on either platform.
The support for macOS as a host is experimental.

### Linux

- systemd.
  Or optionally see [here](#not-using-systemd-on-the-host) for an experimental alternative.
- make
- Docker 1.13.1
- A recent kernel that supports Overlay FS
- git

### Mac

Mac support is experimental.
For a supported configuration on macOS, use Vagrant.

- [Docker for Mac](https://docs.docker.com/docker-for-mac/) (17.03.1 tested).
  See "Graphdriver/Storage driver"
- git

### Vagrant

- [VirtualBox](https://www.virtualbox.org/wiki/Downloads) 5.1.18
- [Vagrant](https://www.vagrantup.com/) 1.9.3
- git

## Setup

**The following steps are REQUIRED on all hosts.**

1. Clone this repo

    ```
    git clone https://github.com/dcos/dcos-docker
    cd dcos-docker
    ```

1. Download [DC/OS](https://dcos.io/releases/) or [Enterprise DC/OS](https://mesosphere.com/product/)
1. Move the installer to `dcos_generate_config.sh` in the root of this repo directory.

**The following steps required if using Vagrant.**

1. (Optional) Install vagrant-vbguest plugin (auto-updates vbox additions)

    ```console
    vagrant plugin install vagrant-vbguest
    ```

1. Bring up the Virtual Machine with a chosen disk size

    Vagrant disks are sparse and as such they will only use the space that is actually used.

    DC/OS should deploy with a size of 100GB, but for larger deployments you may need to increase the size of the VM.

    The first argument is the desired disk size in MB (ex: 102400 is 100GB).

    ```console
    vagrant/resize-disk.sh 102400
    ```

1. SSH into the virtual machine

    ```console
    vagrant ssh
    ```

## Deploy

1. Deploy DC/OS in Docker

    ```console
    make
    ```

1. (Optional) Wait for DC/OS to come up

    ```console
    make postflight
    ```

For other make commands, see `make help`.

## Network Routing

By default in macOS or on either macOS or Linux using Vagrant,
containers are not reachable from the host. This is a problem, for
example, when you want to SSH into a container (not `docker exec`)
or to view the UI from a browser.

### Vagrant

#### Set up routing

To make the Docker containers in the VM reachable from the host, you can route Docker's IP subnet (`172.17.0.0/16`) through the VM's IP (`192.168.65.50`). This routing is not required if you deployed DC/OS to Docker on a native Linux host.

On Linux:
```console
host$ sudo ip route replace 172.17.0.0/16 via 192.168.65.50
```

On macOS:
```console
host$ sudo route -nv add -net 172.17.0.0/16 192.168.65.50
```

Once routing is set up, you can access DC/OS directly from the host.

#### Network Routing Cleanup

On Linux:
```console
host$ sudo ip route del 172.17.0.0/16
```

On macOS:
```console
host$ sudo route delete 172.17.0.0/16
```

Once routing is set up, you can access DC/OS directly from the host.

### Docker for Mac

There are various solutions to allow Docker for Mac containers to be reachable from the host.

[docker-mac-network](https://github.com/wojas/docker-mac-network) has been tested to work.

[Docker for Mac - Host Bridge](https://github.com/mal/docker-for-mac-host-bridge) may also work.

### Node Shell Access

With network routing configured, you can SSH directly into DC/OS nodes from the host:

```console
host$ ssh -i genconf/ssh_key root@172.17.0.2
```

Or you could use the DC/OS CLI:

```console
dcos node ssh --leader --user=root --option IdentityFile=genconf/ssh_key
```

From the Linux host (or SSH'd into Vagrant) you can also use Docker exec to open a shell:

```console
$ docker ps --format="table {{.ID}}\t{{.Names}}\t{{.Status}}"
CONTAINER ID        NAMES                   STATUS
7498dcbe4e3e        dcos-docker-pubagent1   Up About a minute
b66175f0a18a        dcos-docker-agent1      Up About a minute
e80466ce71c9        dcos-docker-master1     Up About a minute

$ docker exec -it dcos-docker-master1 bash
```

## Graphdriver/Storage driver

There is no requirement on the hosts storage driver type, but the docker daemon
running inside docker container supports only `aufs` and `overlay`. The loopback
devicemapper may be problematic when it comes to loopback devices - they may not
be properly cleaned up and thus prevent docker daemon from starting. YMMV
though.

Unless user specifies graphdriver using `DOCKER_GRAPHDRIVER` env variable,
the script tries to use the same one as the host uses. It detects it using
`docker info` command. The resulting graphdriver must be among supported ones,
or the script will terminate.

To check the current storage driver, use `docker info --format "{{json .Driver}}"`.

On Docker for Mac, the default driver is `overlay2`, which is not supported.
Therefore, it is necessary to either set `DOCKER_GRAPHDRIVER` or to change the
host storage driver.

To change the storage driver on Docker for Mac to `overlay`, go to Docker >
Preferences > Daemon Advanced and add `"storage-driver" : "overlay"` to the
configuration file.  Then click "Apply & Restart".

## Settings

### Changing the number of masters or agents

This defaults to 1 master and 1 agent. You can change the number of masters by
setting the variable `MASTERS`. You can change the number of agents by setting
the variable `AGENTS`. For example:

```console
$ make MASTERS=3 AGENTS=5
# start a cluster with 3 masters and 5 agents
```

### Changing the distro

> **NOTE:** This feature should only be used for testing, it is unstable.

By default the cluster will be spun up using a centos base image but if you
want to test something else you can run:

```console
$ make DISTRO=fedora
```

### Not using `systemd` on the host

By default, systemd is used on the host to create a [systemd
slice](https://www.freedesktop.org/software/systemd/man/systemd.slice.html).
This is the supported configuration.

It is possible to run DC/OS Docker on hosts without systemd.  Set the variable
`MESOS_SYSTEMD_ENABLE_SUPPORT` to `false` to disable systemd on the host. This
changes a Mesos setting. Although this setting works at the time of writing, it is not officially supported by DC/OS and so this feature is experimental.

One problem which may occur when not using `systemd` on the host is that executors and tasks will be killed when the agent is restarted. [A JIRA issue](https://jira.mesosphere.com/browse/DCOS_OSS-1131) tracks making it possible to run DC/OS Docker in a supported manner without `systemd`.

## Troubleshooting

Oh dear, you must be in an unfortunate position. You have a few options with
regard to debugging your container cluster.

If the containers are currently running then the best option is to `docker exec`
into the master or agent and poke around. Here is an example of that:

```console
$ docker exec -it dcos-docker-master1 bash

# list the systemd units
[root@dcos-docker-master1 /]# systemctl list-units
...
dbus.socket                         loaded active     running         D-Bus System Message Bus Socket
systemd-fail.service                loaded failed     exited          Journal Audit Socket
systemd-journald-dev-log.socket     loaded active     running         Journal Socket (/dev/log)
systemd-journald.socket             loaded active     running         Journal Socket
basic.target                        loaded active     active          Basic System
dcos.target                         loaded active     active          dcos.target
local-fs.target                     loaded active     active          Local File Systems
...

# find the failed unit and get the status
[root@dcos-docker-master1 /]# systemctl status systemd-fail

# get the logs from journald
[root@dcos-docker-master1 /]# journalctl -xefu systemd-fail
```

For the `dcos-spartan` service to start successfully, make sure that
you have dummy net driver support (`CONFIG_DUMMY`) enabled in your kernel.
Most standard distribution kernels should have this by default. On some
older kernels you may need to manually install this module with
`modprobe dummy` before starting the container cluster.

# Docker out of space

```
docker volume prune
```

## Github Pull Request (PR) Labels

Various labels used on pull requests and what they mean

 - `Work in progress` The code is a work in progress / not yet ready to be
   reviewed or acted upon by others. It can be handy to open up a PR in order
   to share work / ideas with others. Use this label to indicate the PR isn't
   intended to be reviewed or merged.
 - `Request for comment` The code is some idea which may or may not land, but
    there are questions if the approach is right. Review should focus on
    whether or not it is overall a good idea to do this and how to structure it.
 - `Ready for review` The author thinks the PR is ready to land, and is looking for a
    review in order to get it in. The PR may bounce back to "work in progress"
    or "request for comment" if it needs more work or discussion. Might also
    just do all the review and fixup with the label attached.
