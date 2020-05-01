# common.mk

An opiniated common Makefile for all your projects.

* [Why should I need this?](#why-should-i-need-this)
* [How does this work?](#how-does-this-work)
* [How to install this?](#how-to-install-this)
* [We said opiniated](#we-said-opiniated)
  * [common-custom.mk](#common-custommk)
  * [docker-compose-dev.yml](#docker-compose-devyml)
  * [Per service/project Makefile](#per-serviceproject-makefile)
  * [dev.mk](#devmk)
  * [Per service/project .gitignore](#per-serviceproject-gitignore)
  * [dev.Dockerfile](#devdockerfile)
  * [prod.Dockerfile](#proddockerfile)
* [How to use this with your CI?](#how-to-use-this-with-your-ci)


## Why should I need this?

- Let's say you need to handle multiple (micro) services, most probably not all written in the same language.
- Let's say you are the devops in charge of maintaining the pipeline for buidling everything.
- Let's say you don't want to install Java in your CI nodes to build the API, or worse: install NodeJS on your laptop to
  build the front.
- Let's say you want **one** way of building **any** project, without having to install anything locally.

Well, this is for you.


## How does this work?

Multiple targets are defined in this common Makefile (i.e.: `dev`, `build`, `image`, ...) which will all run stuff
inside a Docker container (called "dev"). This "dev" container will be equipped with all the tools your particular
project need so you don't have to install everything locally. Bonus: your CI won't either ;-)

You define inside a `dev.Dockerfile` what you want this container to have.

Then, this dev container will be used for every action (i.e.: compilation, linting, generating code, ...).

You define inside a `dev.mk` what you want to do for every target. This `dev.mk` will automatically be called through
the dev container thanks to `common.mk`. Because the dev container will always be run with the project mounted as a
volume, it will have access to all the code and the artifacts it produces will be available locally.

Example: you have a Java project and you want to build the Jar locally. `cd` in your project, run `make build`: this
will use the `build` target from the `common.mk` which will build the dev container for this project, then run the
`build` target of the `dev.mk` file through the container (which will probably be something like `mvn package`) and
BOOM here is your Jar file right next to your code.

Another Dockerfile (`prod.Dockerfile`) will be needed to build a production-grade Docker image for your service. This
image is built with the `image` target.


## How to install this?

Download the `common.mk` file at the **root** of your project and create a few files in every service/project you want
to make use of `common.mk` (see below for details about these files).

Even though you don't need your entire projects' stack on you laptop for your services/projects builds to work, you
still need these tools:

- **docker**
- **docker-compose**
- **make**
- **git**

That's it. Everything else will be in the dev container.


## We said opiniated

**This will impose you a certain way of building things and a certain layout for your code.**

Let's say you have a mono repo with one Java service (your API), a JavaScript project (your SPA) and a Python project
(your BI / AI).

This is how your repo should look like for `common.mk` to be able to work:

```
myproject/
├── ai/
│   ├── dev.Dockerfile
│   ├── dev.mk
│   ├── .gitignore
│   ├── Makefile
│   ├── prod.Dockerfile
│   └── src/
│       ├── server.py
│       └── ...
├── api/
│   ├── dev.Dockerfile
│   ├── dev.mk
│   ├── .gitignore
│   ├── Makefile
│   ├── prod.Dockerfile
│   └── src/
│       └── main/
│           └── java/
│               └── ...
├── common-custom.mk
├── common.mk
├── docker-compose-dev.yml
└── spa/
    ├── dev.Dockerfile
    ├── dev.mk
    ├── .gitignore
    ├── Makefile
    ├── prod.Dockerfile
    └── src/
        ├── App.js
        └── ...
```

Before diving into the files you have to add to your services/projects, let's see what targets `common.mk` brings to
you, your team and your CI:

- **build**: this target will build your code (be it compilation, transpilation, minification, simple rsync for
  interpreted code...)
- **clean**: clean your source tree from build artifacts
- **dev**: run locally a development version of your service/project
- **help** (the default target run if no target is given to Make): prints help
- **image**: builds the production Docker image of your service/project
- **push**: pushes the production image to the registry you configured
- **tests**: run tests for your project/service

Note that building any service/project will put built files (binary or source files if interpreted, production
Dockerfile, configuration files, ...) in the `build/` folder.

Anything inside the `build/` folder **is supposed to go** in the production image.
Anything that should go in the production image **must be** in the `build/` folder (the `build/` folder is the only
context sent to the Docker daemon).

You can add any other target you like; see [common-custom.mk](#common-custommk) and
[Per service/project Makefile](#per-serviceproject-makefile) below.

Let's now dive into the files you will need:

### common-custom.mk

This file will configure `common.mk` for your particular project:

```Makefile
REGISTRY_URL = <your registry URL>  # i.e.: registry.service.consul
REGISTRY_IMAGE_PREFIX = <your project namespace in this registry>  # i.e.: myproject
```

When the dev container or the prod container will be built, they will be tagged locally as
`<REGISTRY_IMAGE_PREFIX>/<SERVICE_NAME>:<VERSION>` (`VERSION` being `dev` for the dev container and the commit ID and
the current branch for the prod container).

When the prod container will be pushed to the registry, it will be pushed to
`<REGISTRY_URL>/<REGISTRY_IMAGE_PREFIX>/<SERVICE_NAME>:<VERSION>` (`VERSION` being the commit ID and the current
branch - or the current tag if you are on a tag when pushing the image).

You can also use this file to add a new target for all your services/projects at once, i.e.:

```Makefile
.PHONY: grpc

REGISTRY_URL = https://hub.docker.com/r
REGISTRY_IMAGE_PREFIX = myproject

grpc: _dev_image
	$(call container_make,grpc)
```

You can now call `make grpc` in all your services/projects. You still need to tell Make what to do in the `grpc`
target; see [dev.mk](#devmk) below.


### docker-compose-dev.yml

Because we want to keep the makefiles clean and as simple as possible, we will offload the configuration of the dev
build to Docker Compose; i.e.:

```yaml
version: "3.3"
services:

  ai:
    container_name: ai
    image: "myproject/ai:dev"
    command: ["make", "--file", "dev.mk", "dev"]
    volumes:
      - ".:/repo/"
    environment:
      API_HOST: "api"
      API_PORT: "8080"
    build:
      context: ./ai
      dockerfile: dev.Dockerfile
      args:
        USER_ID: "${USER_ID:?}"

  api:
    container_name: api
    image: "myproject/api:dev"
    command: ["make", "--file", "dev.mk", "dev"]
    volumes:
      - ".:/repo/"
    environment:
      DB_HOST: "db"  # see at the bottom of the file
      DB_PORT: "5432"
    ports:
      - 8080:8080
    build:
      context: ./api
      dockerfile: dev.Dockerfile
      args:
        USER_ID: "${USER_ID:?}"

  spa:
    container_name: spa
    image: "myproject/spa:dev"
    command: ["make", "--file", "dev.mk", "dev"]
    volumes:
      - ".:/repo/"
    ports:
      - 8000:8000
    build:
      context: ./spa
      dockerfile: dev.Dockerfile
      args:
        USER_ID: "${USER_ID:?}"

  db:
    container_name: db
    image: "postgres:11.4"
    environment:
      POSTGRES_PASSWORD: "password"
      POSTGRES_USER: "user"
    ports:
      - 5432:5432
    volumes:
      - "db:/var/lib/postgresql/data"
```

Docker Compose allows us to build everything in parallel and spawn the entire dev stack simply and efficiently.


### Per service/project Makefile

Every service/project can define a `Makefile` which have to contain **at least** (i.e.: for the `api` service):

```Makefile
SERVICE_NAME = api  # note that this value is equal to the folder name and the service defined in the docker-compose-dev.yml

include ../common.mk
```

Note that you can add any other target you like. These targets can use the `$(call container_make,<target>)` Make call
to make use of the `common.mk` powerful and streamlined buildchain.

Note also that you can override the `HELP_CONTENT` Make variable to add documentation for this new targets (even though
you'll have to re-declare the whole help content with your content added to it - there is no way to simply extend the
help content right now).


### dev.mk

This file will hold all your commands which are needed to lint, compile, check, ... your source files. Because this
file will always be called through the dev container, all your dev tools will be available when this will run.

i.e. for the `api` service:

```Makefile
.PHONY: build dev tests

# Set the cache folder in the Docker volume so it's not deleted between runs
MAVEN_OPTS = -Dmaven.repo.local=/repo/api/.m2
export MAVEN_OPTS

SRC_FILES = $(shell find src/main/ -type f -name '*.java') pom.xml $(shell find src/main/resources -type f)

build/api-jar-with-dependencies.jar: ${SRC_FILES}
	mvn clean package

build: build/api-jar-with-dependencies.jar
	cp prod.Dockerfile build/Dockerfile

dev: build
	LOG_LEVEL=debug java -jar build/api-jar-with-dependencies.jar

tests:
	mvn tests
```

Note that if you already have all the tools installed locally (because you're a developper for example), you can still
use this `dev.mk` file to simplify your day-to-day workflow (you could run `make -f dev.mk dev` instead of the more
complex full Java call).


### Per service/project .gitignore

Because the `build/` folder will only contain all your build artifacts, it should be ignored by git. Every
service/project should have a `.gitignore` file with at least this:

```gitignore
/build/
```


### dev.Dockerfile

This Dockerfile will be used to create the dev container, so it needs to ship everything your project need for
development / build.

i.e. for the `api` service:

```Dockerfile
FROM fedora:30

ARG USER_ID
RUN adduser --create-home --uid ${USER_ID} dev  # see below why this is useful

RUN dnf install -y \
		java-1.8.0-openjdk \
		make \
		maven-3.5.4 \
	&& dnf clean all

WORKDIR /repo/api
USER dev

CMD ["/bin/bash"]
```

**The dev container will always run with the same PID as the user invoking it**, so the process inside your container
can access all your files (because for the kernel, it will be you) and it won't mess with the ownership of your project
files.

You can use any base container you like, any dependencies / tools. **You just have to ship make in the dev container**
(because `common.mk` will call `make` for every action through the dev container).


### prod.Dockerfile

This Dockerfile will be the Dockerfile used to build the production-grade container for your service. You should make
it as light as possible. While the dev container always stays on the local machine, the prod container will be pushed
to a registry and possibly pulled by dozens (hundreds?) of nodes.

Please be nice to the bandwidth and make it as small as possible.

i.e. for the `api` service:

```Dockerfile
FROM alpine:3.10.1

RUN apk add --no-cache openjdk8

RUN adduser api -D -s /sbin/nologin

COPY --chown=api:api api-jar-with-dependencies.jar /home/api/

USER api

CMD ["java", "-jar", "/home/api/api-jar-with-dependencies.jar"]
```

Note that this file should be simply copied in the build folder by the build target of the `dev.mk`.


## How to use this with your CI?

`common.mk` has been thought to be used by devops, developers and CI/CD.

No matter who build your service/project, the build will always be the same. `common.mk` aims at reproductibility and
consistency thanks to containers.

If you have a CI, you don't need to redefine how to build your services/projects and you don't need to install all the
tooling on your CI nodes. You just have to install `docker` and `docker-compose` (and `git` and `make` but there is a
99% chance they are already installed).

For example, your CI pipeline on Jenkins for the services defined in this file could be (note that this Jenkins
pipeline makes use of parallelism but you don't have to):

```groovy
pipeline {
	agent any

	environment {
		CI = "true"  # this is needed so common.mk will use non-interactive tools
	}

	stages {
		stage("myproject") {
			parallel {
				stage("api") {
					steps {
						sh "make --directory=api tests"
						sh "make --directory=api build"
						sh "make --directory=api image"
						sh "make --directory=api push"
					}
				}
				stage("ai") {
					steps {
						sh "make --directory=ai tests"
						sh "make --directory=ai build"
						sh "make --directory=ai image"
						sh "make --directory=ai push"
					}
				}
				stage("spa") {
					steps {
						sh "make --directory=spa tests"
						sh "make --directory=spa build"
						sh "make --directory=spa image"
						sh "make --directory=spa push"
					}
				}
			}
		}
	}
}
```
