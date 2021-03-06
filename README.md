ckan-docker
===========

Developing and deploying CKAN with Docker



# Intro

Dockerfiles, Fig service definition & Vagrantfile to develop & deploy CKAN, Postgres, Solr & datapusher using Docker.

Docker containers included:

- CKAN (should work with any version 2.x)
- Postgres (Postgres 9.3 and PostGIS 2.1, CKAN datastore & spatial extension supported)
- Solr (4.10.1, custom schemas & spatial extension supported)
- Fig _[optional]_ (to manage the containers)
- Data _[optional]_ (to store Postgres data & CKAN FileStore separately)


Other contrib containers:

- Nginx (1.7.6 official) as a caching reverse proxy
- Datapusher


# Requirements

|Name			|Version		|Comment										|
|:--------------|:-------------:|:----------------------------------------------|
|Docker			|>= 1.3 		|works with Boot2docker 1.3						|
|Fig 			|>= 1.0 		|on the host or with Dockerfile provided		|
|Vagrant		|>= 1.6 		|if you intend to use Vagrant					|
|OS				|any	 		|as long as you can run Docker 1.3				|



# Reference

## Structure

	├── Dockerfile (CKAN Dockerfile)
	├── README.md
	├── Vagrantfile (CKAN Vagrantfile)
	├── _etc (config copied to /etc)
	│   ├── apache2
	│   ├── ckan
	│   ├── cron.d
	│   ├── my_init.d
	│   ├── postfix
	│   └── supervisor
	├── _service-provider (any service provider such as datapusher)
	│   └── datapusher
	├── _solr
	│   └── schema.xml (version specific & custom schema)
	├── _src (CKAN source code & extensions)
	│   ├── ckan
	│   └── ckanext-...
	├── docker
	│   ├── ckan
	│   ├── data
	│   ├── fig
	│   ├── nginx
	│   ├── insecure_key (baseimage insecure SSH key)
	│   ├── postgres
	│   └── solr
	├── fig.yml (CKAN services definition)
	└── vagrant
	    └── docker-host (Linux Docker host if required)

### Directories

_the content from the directories prefixed with `_` need to be edited / configured as required before building the Dockerfiles._

#### _etc
contains configuration files that are copied to /etc in the container. see _etc/README

#### _solr
contains your custom Solr schema (for your version of CKAN, & extensions installed). see _solr/README

#### _src
contains your packages source code (CKAN & extensions). see _src/README

#### _service-provider
contains any service providers (e.g. datapusher) with their Dockerfiles. see _service-provider/README.

#### docker
contains the Dockerfiles and any supporting files

#### vagrant
contains the Docker host if the host cannot run Docker containers natively (OS X & Windows)


### Files

#### Dockerfiles

The Dockerfiles are currently based on `phusion/baseimage:0.9.16`.

SSH is supported using an insecure key which is enabled by default for development purposes. You should disable it in production use for obvious reasons.

[Read this to find out more about phusion baseimage](https://phusion.github.io/baseimage-docker/)

##### CKAN Dockerfile
The app container runs the following services

- Apache
- Postfix
- Supervisor
- Cron

##### Postgres Dockerfile
The database container runs Postgres 9.3 and PostGIS 2.1.
It supports the [datastore](http://docs.ckan.org/en/latest/maintaining/datastore.html) & [ckanext-spatial](https://github.com/ckan/ckanext-spatial)

##### Data Dockerfile
The data container is optional but recommended. It exposes two volumes to store the Postgres data `($PGDATA)` & CKAN FileStore. This means you can recreate / app containers without losing your data.

##### Solr Dockerfile
The Solr container runs version 4.10.1. This can easily be changed by customising SOLR_VERSION in the Dockerfile.

By detault the `schema.xml` of the upstream version (2.3) is copied in the container. This can be overriden at runtime by mounting it as a volume.
This default path of the volume is `<path to>/_src/ckan/ckan/config/solr/schema.xml` so it mounts the schema corresponding to your version of CKAN.

For example for Fig:

    solr:
      build: docker/solr
      hostname: solr
      domainname: localdomain
      ports:
        - "8983:8983"
      volumes:
        - <path to>/_src/ckan/ckan/config/solr/schema.xml:/opt/solr/example/solr/ckan/conf/schema.xml

If you need a custom schema, put it in `<full path to>/_solr` and change the path in the fig or vagrant file.

      volumes:
        - <path to>/_solr/schema.xml:/opt/solr/example/solr/ckan/conf/schema.xml


The container is cross version compatible. You need mount the appropriate `schema.xml` as a volume, or build a child image, which will copy the `schema.xml` next to your Dockerfile.

Read the [ckanext-spatial documentation](http://docs.ckan.org/projects/ckanext-spatial/en) to add the required fields to your Solr schema if you use ckanext-spatial


##### Fig Dockerfile
The Fig container runs Fig version `1.0.1` & the latest Docker within a container.

The Docker socket needs to be mounted as a volume to control Docker on the host. A source folder must be mounted to access the fig definition

see docker/Fig/Readme to find out how to use

#### Vagrantfile
Defines VMs provided by Docker, a Virtual Box docker-host is used if the host can't run Docker containers natively. This is an alternative to Boot2Docker.

#### fig.yml
Defines the set of services required to run CKAN. Read the [fig.yml reference](http://www.fig.sh/yml.html) to understand and edit.



---
# Usage

1. Clone your code in the `_src` directory (see _src/README)
2. Clone the datapusher in `_service-provider` (see _service-provider/README)
3. Set the full path of the volumes in fig.yml
4. Run `up` with Fig or Vagrant


## Using Fig (recommended)

#### Option 1: Fig is installed on the Docker host
_If you have if >= 1.0 installed, just type_

	fig up

#### Option 2: Using the fig container
_Otherwise, you can use the container provided_

Build fig the fig container

	docker build --tag="fig_container" docker/fig

Run it

	docker run -it -d --name="fig-ckan" -p 2375 -v /var/run/docker.sock:/tmp/docker.sock -v $(pwd):/src fig_container

_In the fig container fig won't work with relative path, because the mount namespace is different, you need to change the relative path to absolute path_

for example, change the `./`:

	volumes:
	    - ./_src:/usr/lib/ckan/default/src

to an absolute path  to you ckan-docker directory: `/Users/username/git/ckan/ckan-docker/`

	volumes:
	    - /Users/username/git/ckan/ckan-docker/_src:/usr/lib/ckan/default/src

Build & Run the services defined in `fig.yml`

	docker exec -it fig-ckan fig up

If you are using boot2docker, add entries in your hosts file e.g. `192.168.59.103  ckan.localdomain`

You can now access CKAN at http://ckan.localdomain:8080/ (Apache) & http://ckan.localdomain/ (Ngnix)

## Using Vagrant

Build & run

	vagrant up --provider=docker --no-parallel

You can now access CKAN at http://localhost:8080/ (Apache)

You can also SSH inside the container if you have left the `--enable-insecure-key` option in the run command.

	vagrant ssh ckan

SSH insecure key can be disabled by removing the `--enable-insecure-key` option from the run command.

---
## Running commands inside the container

The simplest thing to do is to use the `docker exec` command, for example:

	docker exec -it src_ckan_1 /bin/bash

You can also SSH inside the container if you have left the `--enable-insecure-key` option in the run command.

	ssh -i docker/insecure_key -p 2222 root@ckan.localdomain

SSH insecure key can be disabled by removing the `--enable-insecure-key` option from the run command.

## Managing Docker images & containers

You should use fig to manage your containers & images, this will ensure they are started/stopped in order

If you want to quickly remove all untagged images:

	docker images -q --filter "dangling=true" | xargs docker rmi

If you want to quickly remove all stopped containers

	docker rm $(docker ps -a -q)

---
## Developing CKAN

### Using paster serve instead of apache for development
CKAN container starts Apache2 by default and the `ckan.site_url` port is set to `8080` in `50_configure`.
You can override that permanently in the `custom_options.ini`, or manually in the container, for instance if you want to use paster in a development context.

Example (`paster serve --reload` in debug mode):

	docker exec -it src_ckan_1 /bin/bash
	supervisorctl stop apache2
	sed -i -r 's/debug = false/debug = true/' $CKAN_CONFIG/$CONFIG_FILE
	sed -i -r 's/ckan.localdomain:8080/ckan.localdomain:5000/' $CKAN_CONFIG/$CONFIG_FILE
	$CKAN_HOME/bin/paster serve --reload $CKAN_CONFIG/$CONFIG_FILE

### Frontend development
Front end development is also possible (see [Frontend development guidelines](http://docs.ckan.org/en/latest/contributing/frontend/))

Install frontend dependencies:

	docker exec -it src_ckan_1 /bin/bash
	apt-get update
	apt-get install -y nodejs npm
	ln -s /usr/bin/nodejs /usr/bin/node
	source $CKAN_HOME/bin/activate
	cd $CKAN_HOME/
	npm install nodewatch less@1.3.3

Both examples show that development dependencies should only be installed in the containers when required. Since they are not part of the `Dockerfile` they do not persist and only serve the purpose of development. When they are no longuer needed the container can be rebuilt allowing to test the application in a production-like state.

---

# Sources
- [Docker](https://www.docker.com)
- [Fig](http://www.fig.sh)
- [Vagrant Docker provider](https://docs.vagrantup.com/v2/docker/index.html)
