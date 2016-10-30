[![Donate](https://img.shields.io/badge/Donate-PayPal-blue.svg)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=WZJTZ3V8KKARC) [![Fork on GitHub](https://img.shields.io/github/forks/badges/shields.svg?style=flat&label=Fork%20on%20GitHub&color=blue)](https://github.com/JensPiegsa/docker-cheat-sheet/edit/master/README.md#fork-destination-box)

This document is hosted at [https://jenspiegsa.github.io/docker-cheat-sheet/](https://jenspiegsa.github.io/docker-cheat-sheet/).

# Content

* [1. Fundamentals](#1-fundamentals)
	* [1.1. Concepts](#11-concepts)
	* [1.2. Lifecycle](#12-lifecycle)
* [2. Recipes](#2-recipes)
	* [2.1. Docker Engine](#21-docker-engine)
		* [2.1.1. Building Images](#211-building-images)
		* [2.1.2. Running Containers](#212-running-containers)
		* [2.1.3. Using Volumes](#213-using-volumes)
	* [2.2. Docker Machine](#22-docker-machine)
	* [2.3. Dockerfile](#23-dockerfile)
* [3. Best Practices](#3-best-practices)
* [4. Additional Material](#4-additional-material)

# 1. Fundamentals

# 1.1. Concepts

* **Union file system (UFS)** allows to overlay multiple file systems appearing as a single esytem whereby equal folders are merged and equally named files hide their previous versions
* **Image** a portable read-only file system layer optionally stacked on a parent image
* **Dockerfile** used to `build` an image and declare the command executed in the container
* **Registry** is the place where to `push` and `pull` from named / tagged images 
* **Container** an instance of an image with a writable file system layer on top, virtual networking, ready to execute a single application 
* **Volume** a specially-designated directory outside the UFS for persistent and shared data that can be mounted inside containers
* **Network** acts as a namespace for containers
* **Service** a flexible number of container replicas running on a cluster of multiple hosts

# 1.2. Lifecycle

**A typical `docker` workflow:**

* `build` an image based on a `Dockerfile`
* `tag` and `push` the image to a **registry**
* `login` to the registry from the runtime environment to `pull` the image
* optionally `create` a `volume` or two to provide configuration files and hold data that needs to be persisted 
* `run` a container based on the image
* `stop` and `start` the container if necessary
* `commit` the container to turn it into an image
* in exceptional situations, `exec` additional commands inside the container
* to replace a container with an updated version 
	* `pull` the new image from the registry
	* `stop` the running container
	* backup your volumes to prepare a rollback
	* `run` the newer one by specifying a temporary name
	* if successful, `remove` the old container and `rename` the new one accordingly
 
# 2. Recipes

## 2.1. Docker Engine

### 2.1.1. Building Images

#### Debug image build

* `docker build` shows the IDs of all temporary containers and intermediate images
* use `docker run -it IMAGE_ID` with the ID of the image resulting from the last successful build step and try the next command manually

### 2.1.2. Running Containers

#### Start container and run command inside

```sh
docker run -it ubuntu:14.04 /bin/bash
```

#### Start a shell in a running container

```sh
docker exec -it CONTAINER /bin/bash
```

#### Start a container with another user

```sh
docker run -u root IMAGE
```

#### List all existing containers

```sh
docker ps -a
```

#### List running processes inside a container

```sh
docker top CONTAINER
```
     
#### Follow the logs

```sh
docker -f --tail=1000 CONTAINER
```

#### Stop all running containers

```sh
docker stop $(docker ps -q)
```

#### Remove all stopped containers, except those suffixed '-data':


```sh
docker ps -a -f status=exited | grep -v '\-data *$'| awk '{if(NR>1) print $1}' | xargs -r docker rm
```

#### Remove all stopped containers (warning: removes data-only containers too)

```sh
docker rm $(docker ps -qa -f status=exited)
```

* *note: the filter flag `-f status=exited` may be omitted here since running containers can not be removed*

#### List all images

```sh
docker images -a
```

#### Remove all unused images

```sh
docker rmi $(docker images -qa -f dangling=true)
```

#### Show image history of container

```sh
docker history --no-trunc=true $(docker inspect -f '{{.Image}}' CONTAINER)
```

#### Show file system changes compared to the original image

```sh
docker diff CONTAINER
```

#### Backup volume to host directory

```sh
docker run -rm --volumes-from SOURCE_CONTAINER -v $(pwd):/backup busybox \
 tar cvf /backup/backup.tar /data
```

#### Restore volume from host directory

```sh
docker run -rm --volumes-from TARGET_CONTAINER -v $(pwd):/backup busybox tar xvf /backup/backup.tar
```

#### Show volumes

```sh
docker inspect -f '{{range $v, $h := .Config.Volumes}}{{$v}}{{end}}' CONTAINER
```

#### Start all paused / stopped containers

* makes no sense together with container dependencies

#### Remove all containers and images

```sh
docker stop $(docker ps -q) && docker rm $(docker ps -qa) && docker rmi $(docker images -qa)
```

#### Edit and update a file in a container

```sh
docker cp CONTAINER:FILE /tmp/ && docker run --name=nano -it --rm -v /tmp:/tmp \
 piegsaj/nano nano /tmp/FILE ; \
cat /tmp/FILE | docker exec -i CONTAINER sh -c 'cat > FILE' ; \
rm /tmp/FILE
```

#### Deploy war file to Apache Tomcat server instantly

```sh
docker run -i -t -p 80:8080 -e WAR_URL=“<http://web-actions.googlecode.com/files/helloworld.war>” \
 bbytes/tomcat7
```

#### Dump a Postgres database into current directory on the host

```sh
echo "postgres_password" | sudo docker run -i --rm --link db:db -v $PWD:/tmp postgres:8 sh -c ' \
 pg_dump -h ocdb -p $OCDB_PORT_5432_TCP_PORT -U postgres -F tar -v openclinica \
 > /tmp/ocdb_pg_dump_$(date +%Y-%m-%d_%H-%M-%S).tar'
```

#### Backup data folder

```sh
docker run --rm --volumes-from oc-data -v $PWD:/tmp piegsaj/openclinica \
 tar cvf /tmp/oc_data_backup_$(date +%Y-%m-%d_%H-%M-%S).tar /tomcat/openclinica.data
```

#### Restore volume from data-only container

```sh
docker run --rm --volumes-from oc-data2 -v $pwd:/tmp piegsaj/openclinica \
 tar xvf /tmp/oc_data_backup_*.tar
```

#### Get the IP address of a container

```sh
docker inspect container_id | grep IPAddress | cut -d '"' -f 4
```

## 2.1.3. Using Volumes

#### Declare a volume via Dockerfile

```
RUN mkdir /data && echo "some content" > /data/file && chown -R daemon:daemon /data
VOLUME /data
```

* *note: after the `VOLUME` directive, its content can not be changed within the Dockerfile*

#### Create a volume at runtime

```sh
docker run -it -v /data debian /bin/bash
```

#### Create a volume at runtime bound to a host directory

```sh
docker run --rm -v /tmp:/data debian ls -RAlph /data
```

#### Create a named volume and use it

```sh
docker volume create --name=test
docker run --rm -v test:/data alpine sh -c 'echo "Hello named volumes" > /data/hello.txt'
docker run --rm -v test:/data alpine sh -c 'cat /data/hello.txt'
```

#### List the content of a volume

```sh
docker run --rm -v data:/data alpine ls -RAlph /data
```

#### Copy a file from host to named volume

```sh
echo "debug=true" > test.cnf && \
docker volume create --name=conf && \
docker run --rm -it -v $(pwd):/src -v conf:/dest alpine cp /src/test.cnf /dest/ && \
rm -f test.cnf && \
docker run --rm -it -v conf:/data alpine cat /data/test.cnf
```

#### Copy content of existing named volume to a new named volume

```sh
docker volume create --name vol_b
docker run --rm -v vol_a:/source/folder -v vol_b:/target/folder -it \
 rawmind/alpine-base:0.3.4 cp -r /source/folder /target
```

## 2.2. Docker Machine

### On a local VM

#### Get the IP address of the virtual machine for access from host

```
docker-machine ip default
```

#### Add persistent environment variable to boot2docker

```sh
sudo echo 'echo '\''export ENVTEST="Hello Env!"'\'' > /etc/profile.d/custom.sh' | \
sudo tee -a /var/lib/boot2docker/profile > /dev/null
```

and restart with `docker-machine restart default`

#### Install additional linux packages in boot2docker

* create the file `/var/lib/boot2docker/bootsync.sh` with a content like:

```sh
#!/bin/sh
sudo /bin/su - docker -c 'tce-load -wi nano'
```

#### Recreate any folders and files on boot2docker startup

* store folders / files in `/var/lib/boot2docker/restore-on-boot` and
* create the file `/var/lib/boot2docker/bootsync.sh` with a content like:

```sh
#!/bin/sh
sudo mkdir -p /var/lib/boot2docker/restore-on-boot && \
sudo rsync -a /var/lib/boot2docker/restore-on-boot/ /
```

## 2.3. Dockerfile

#### Add a periodic health check

```
HEALTHCHECK --interval=1m --timeout=3s --retries=5 \
 CMD curl -f <http://localhost/> || exit 1
```

* see also: [HEALTHCHECK](https://docs.docker.com/engine/reference/builder/#/healthcheck)

# 3. Best Practices

## Docker Engine

* `docker exec` is your friend in development, but should be avoided in a production setup

## Volumes

* use *named volumes* to simplify maintenance by separating persistent data from the container and communicating the structure of a project in a more transparent manner

## Dockerfile

* always set the `USER` statement, otherwise the container will run as `root` user by default, which maps to the `root` user of the host machine
* use `ENTRYPOINT` and `CMD` directives together to make container usage more convenient
* combine consecutive `RUN` directives with `&&` to reduce the costs of a build and to avoid caching of instructions like `apt-get update`
* use `EXPOSE` to document all needed ports

# 4. Additional Material

* [Mouat, A. (2015). *Using Docker: Developing and Deploying Software with Containers.* O'Reilly Media.](http://shop.oreilly.com/product/0636920035671.do) ([German Edition: *Docker. Software entwickeln und deployen mit Containern.* dpunkt.verlag](https://www.dpunkt.de/buecher/12553/9783864903847-docker.html))
* [Official Docker Documentation](https://docs.docker.com/)
* [StackOverflow Documentation](http://stackoverflow.com/documentation/docker/topics)
