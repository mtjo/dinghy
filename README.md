# dinghy

Docker on OS X with batteries included, aimed at making a more pleasant local development experience.
Runs on top of [docker-machine](https://github.com/docker/machine).

  * Faster volume sharing using NFS rather than built-in virtualbox/vmware file shares. A medium-sized Rails app boots in 5 seconds, rather than 30 seconds using vmware file sharing, or 90 seconds using virtualbox file sharing.
  * Filesystem events work on mounted volumes. Edit files on your host, and see guard/webpack/etc pick up the changes immediately.
  * Easy access to running containers using built-in DNS and HTTP proxy.

Dinghy creates its own VM using `docker-machine`, it will not modify your existing `docker-machine` VMs.

Eventually `docker-machine` may have a rich enough plugin system that dinghy can
just become a plugin to `docker-machine`. For now, dinghy runs as a wrapper
around `docker-machine`, shelling out to create the VM and using `launchd` to
start the various services such as NFS and DNS.

## FAQ and solutions to common problems

Before filing an issue, see the [FAQ](FAQ.md).

## upgrading from vagrant

If you previously used a version of Dinghy that ran on top of Vagrant, [read this](UPGRADE_FROM_VAGRANT.md).

## install

First the prerequisites:

1. OS X Yosemite (10.10) or higher
1. [Homebrew](https://github.com/Homebrew/homebrew)
1. Docker and Docker Machine. These can either be installed with Homebrew (`brew install docker docker-machine`), or using a package such as the Docker Toolbox.
1. A Virtual Machine provider for Docker Machine. Currently supported options are:
    * [xhyve](http://www.xhyve.org/) installed with [docker-machine-driver-xhyve](https://github.com/zchee/docker-machine-driver-xhyve#install).
    * [VirtualBox](https://www.virtualbox.org). Version 5.0+ is strongly recommended, and you'll need the [VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads) installed.
    * [VMware Fusion](http://www.vmware.com/products/fusion).
    * [Parallels](https://www.parallels.com/products/desktop/) installed with [docker-machine-parallels](https://github.com/Parallels/docker-machine-parallels).

Then:

    $ brew tap codekitchen/dinghy
    $ brew install dinghy

You will need to install `docker` and `docker-machine` as well, either via Homebrew or the official Docker package downloads. To install with Homebrew:

    $ brew install docker docker-machine

You can specify provider (`virtualbox`, `vmware`, `xhyve` or `parallels`), memory and CPU options when creating the VM. See available options:

    $ dinghy help create

Then create the VM and start services with:

    $ dinghy create --provider virtualbox

Once the VM is up, you'll get instructions to add some Docker-related
environment variables, so that your Docker client can contact the Docker
server inside the VM. I'd suggest adding these to your .bashrc or
equivalent.

Sanity check!

    $ docker run -it redis

## CLI Usage

```bash
$ dinghy help
Commands:
  dinghy create          # create the docker-machine VM
  dinghy destroy         # stop and delete all traces of the VM
  dinghy halt            # stop the VM and services
  dinghy help [COMMAND]  # Describe available commands or one specific command
  dinghy ip              # get the VM's IP address
  dinghy restart         # restart the VM and services
  dinghy shellinit       # returns env variables to set, should be run like $(dinghy shellinit)
  dinghy ssh [args...]   # ssh to the VM
  dinghy status          # get VM and services status
  dinghy up              # start the Docker VM and services
  dinghy upgrade         # upgrade the boot2docker VM to the newest available
  dinghy version         # display dinghy version
```

## DNS

Dinghy installs a DNS server listening on the private interface, which
resolves \*.docker to the Dinghy VM. For instance, if you have a running
container that exposes port 3000 to the host, and you like to call it
`myrailsapp`, you can connect to it at `myrailsapp.docker` port 3000, e.g.
`http://myrailsapp.docker:3000/` or `telnet myrailsapp.docker 3000`.

## HTTP proxy

Dinghy will run a HTTP proxy inside a docker container in the VM, giving you
easy access to web apps running in other containers. This is based heavily on
the excellent [nginx-proxy](https://github.com/jwilder/nginx-proxy) docker tool.

The proxy will take a few moments to download the first time you launch the VM.

Any containers that you want proxied, make sure the `VIRTUAL_HOST`
environment variable is set, either with the `-e` option to docker or
the environment hash in docker-compose. For instance setting
`VIRTUAL_HOST=myrailsapp.docker` will make the container's exposed port
available at `http://myrailsapp.docker/`. If the container exposes more
than one port, set `VIRTUAL_PORT` to the http port number, as well.

See the nginx-proxy documentation for further details.

If you use docker-compose, you can add VIRTUAL_HOST to the environment hash in
`docker-compose.yml`, for instance:

```yaml
web:
  build: .
  environment:
    VIRTUAL_HOST: myrailsapp.docker
```

### SSL Support

SSL is supported using single host certificates using naming conventions.

To enable SSL, just put your certificates and privates keys in the ```HOME/.dinghy/certs``` directory
for any virtual hosts in use.  The certificate and keys should be named after the virtual host with a `.crt` and
`.key` extension.  For example, a container with `VIRTUAL_HOST=foo.bar.com.docker` should have a
`foo.bar.com.docker.crt` and `foo.bar.com.docker.key` file in the certs directory.

#### How SSL Support Works

The SSL cipher configuration is based on [mozilla nginx intermediate profile](https://wiki.mozilla.org/Security/Server_Side_TLS#Nginx) which
should provide compatibility with clients back to Firefox 1, Chrome 1, IE 7, Opera 5, Safari 1,
Windows XP IE8, Android 2.3, Java 7.  The configuration also enables HSTS, and SSL
session caches.

The default behavior for the proxy when port 80 and 443 are exposed is as follows:

* If a container has a usable cert, port 80 will redirect to 443 for that container so that HTTPS
is always preferred when available.
* If the container does not have a usable cert, port 80 will be used.

To serve traffic in both SSL and non-SSL modes without redirecting to SSL, you can include the
environment variable `HTTPS_METHOD=noredirect` (the default is `HTTPS_METHOD=redirect`).  You can also
disable the non-SSL site entirely with `HTTPS_METHOD=nohttp`.

#### How to quickly generate self-signed certificates

You can generate self-signed certificates using ```openssl```.

```bash
openssl req -x509 -newkey rsa:2048 -keyout foo.bar.com.docker.key \
-out foo.bar.com.docker.crt -days 365 -nodes \
-subj "/C=US/ST=Oregon/L=Portland/O=Company Name/OU=Org/CN=foo.bar.com.docker"
```

To prevent your browser to emit warning regarding self-signed certificates, you can install them on your system as trusted certificates.

## Preferences

Dinghy creates a preferences file under ```HOME/.dinghy/preferences.yml```, which can be used to override default options. This is an example of the default generated preferenes:

```
:preferences:
  :proxy_disabled: false
  :fsevents_disabled: false
  :create:
    provider: virtualbox
```

If you want to override the dinghy machine name (e.g. to change it to 'default' so it can work with Kitematic), it can be changed here. First, destroy your current dinghy VM and then add the following to your preferences.yml file:

```
:preferences:
.
.
.
  :machine_name: default
```

## a note on NFS sharing

Dinghy shares your home directory (`/Users/<you>`) over NFS, using a
private network interface between your host machine and the Dinghy
Docker Host. This sharing is done using a separate NFS daemon, not the
system NFS daemon.

Be aware that there isn't a lot of security around NFSv3 file shares.
We've tried to lock things down as much as possible (this NFS daemon
doesn't even listen on other interfaces, for example).

## upgrading

If you didn't originally install Dinghy as a tap, you'll need to switch to the
tap to pull in the latest release:

    $ brew tap codekitchen/dinghy

To update Dinghy itself, run:

    $ dinghy halt
    $ brew update
    $ brew upgrade dinghy
    $ dinghy up

To update the Docker VM, run:

    $ dinghy upgrade

This will run `docker-machine upgrade` and then restart the dinghy services.

### prereleases

You can install Dinghy's master branch with:

    $ dinghy halt
    $ brew reinstall --HEAD dinghy
    $ dinghy up

This branch may be less stable, so this isn't recommended in general.

## built on

 - https://github.com/docker/machine
 - https://github.com/markusn/unfs3
 - https://github.com/Homebrew/homebrew
 - http://www.thekelleys.org.uk/dnsmasq/doc.html
 - https://github.com/jwilder/nginx-proxy
