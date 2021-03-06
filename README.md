# Multi-Stage Docker build for Go

## TL;DR

Dockerize your `golang` app easily with the new multi-stage builds from Docker 17.05. Reduce deployment steps and produce smaller, optimized builds.

## Audience:

- You want to know how to Dockerize your `Golang` app
- You want your Docker image to be as small as possible
- You want to know how multi-stage docker build works and the pros

## References:

- The example can be found in the Github repo [here](https://github.com/alextanhongpin/go-docker-multi-stage-build)

## Highlights

- You will first build a docker image using only the `Docker` golang base image, and observe the outcome. For simplicity, our program will just output __"hello, go"__
- Then, you will learn how to build a more optimized docker image, but requires separate commands
- Finally, we will demonstrate how multi-stage build can simplify our process


## Why label-schema.org?

It's important to tag your images with sufficient information because:

1) you want to know where is the source code for the docker build
2) you want to know which branch/git commit it is using
3) you want to know when was the last build
4) you want to know how to rebuild it
5) you want to know who last build it
6) you want to know what this docker image do, how to run it and what is the environment variables

## Guide


## Setup

You need to have `golang` and a minimum version of `docker 17.05` installed in order to run this demo. You can check the version of your dependencies as shown below:

Validating __go__ version:
```bash
$ go version 
go version go1.9 darwin/amd64
```

Validating __Docker__ version:
```bash
$ docker version
Client:
 Version:      17.06.1-ce
 API version:  1.30
 Go version:   go1.8.3
 Git commit:   874a737
 Built:        Thu Aug 17 22:53:38 2017
 OS/Arch:      darwin/amd64

Server:
 Version:      17.06.1-ce
 API version:  1.30 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   874a737
 Built:        Thu Aug 17 22:54:55 2017
 OS/Arch:      linux/amd64
 Experimental: true
```

## The golang program

The `main.go` contains our application logic. It does nothing but print `Hello, go!`.

```go
package main

import "log"

func main() {
	log.Println("Hello, go!")
}
```

Now that we have our application, let's dockerize it!

## Method 1: Using the Golang image

The steps in `Dockerfile.00` is as follow:

1. We select the `golang:1.9` image
2. We create a workdir called hello-world
3. We copy the file into the following directory
4. We get all the dependencies required by our application
5. We compile our application to produce a static binary called `app`
6. We run our binary

```Dockerfile
FROM golang:1.9

WORKDIR /go/src/github.com/alextanhongpin/hello-world

COPY main.go .

RUN go get -d -v

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

CMD ["/go/src/github.com/alextanhongpin/hello-world/app"]
```

Let's build an image called `alextanhongpin/hello-world-00` out of it. You can use your Github username instead when building the image.

```bash
$ docker build -t alextanhongpin/hello-world-00 -f Dockerfile.00 .

Sending build context to Docker daemon  2.016MB
Step 1/6 : FROM golang:1.9
 ---> 5e2f23f821ca
Step 2/6 : WORKDIR /go/src/github.com/alextanhongpin/hello-world
 ---> Using cache
 ---> d36bf8436458
Step 3/6 : COPY main.go .
 ---> Using cache
 ---> 2fa05dc652bc
Step 4/6 : RUN go get -d -v
 ---> Using cache
 ---> bb0f73ac82d1
Step 5/6 : RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
 ---> Using cache
 ---> 8b32d3f4cfd0
Step 6/6 : CMD /go/src/github.com/alextanhongpin/hello-world/app
 ---> Running in 440d47e71346
 ---> 2669fc5303bf
Removing intermediate container 440d47e71346
Successfully built 2669fc5303bf
Successfully tagged alextanhongpin/hello-world-00:latest
```

We will run our docker image to validate that it is working:

```bash
$ docker run alextanhongpin/hello-world-00
Hello, go!
```

Let's take a look at the image size that is produced:

```bash
docker image list | grep hello-world
alextanhongpin/hello-world-00                   latest              2669fc5303bf        42 seconds ago      729MB
```

We have a `729MB` image for a simple `Hello, go!`! What can we do to minimize it? That brings us to next step...

## Method 2: Build locally

The reduce the size, we can try to compile our `main.go` locally and copy the executable to an alpine image - the size should be smaller since it contains only our executable, but without the __go runtime__. Let's compile our `main.go`:

```bash
$ CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
```

`Dockerfile.01` contains the step to build our second image:

```Dockerfile
FROM alpine:latest  
RUN apk --no-cache add ca-certificates

COPY app .

CMD ["/app"]
```

All it does is copy our compiled binary to an __alpine__ image. We will build the image with the following command:

```bash
$ docker build -t alextanhongpin/hello-world-01 -f Dockerfile.01 .

Sending build context to Docker daemon  2.017MB
Step 1/4 : FROM alpine:latest
 ---> 7328f6f8b418
Step 2/4 : RUN apk --no-cache add ca-certificates
 ---> Using cache
 ---> 70fb51eb7cf7
Step 3/4 : COPY app .
 ---> b2a128947460
Removing intermediate container 79ec202de604
Step 4/4 : CMD /app
 ---> Running in fa74b21e353a
 ---> d678076674fa
Removing intermediate container fa74b21e353a
Successfully built d678076674fa
Successfully tagged alextanhongpin/hello-world-01:latest
```

Let's validate it again as we did before and view the change in the size:

```bash
$ docker run alextanhongpin/hello-world-01
Hello, go!
```

Let's take a look at the image size:

```bash
docker image list | grep hello-world
alextanhongpin/hello-world-01                   latest              d678076674fa        45 seconds ago      6.55MB
alextanhongpin/hello-world-00                   latest              2669fc5303bf        5 minutes ago       729MB
```

We can see that the size has reduced dramatically from `729MB` to `6.55MB`. This however, involves two different step - compiling the binary locally and create a docker image. The next section will demonstrate how you can reduce this to a single step. 


## Method 3: Using multi-stage build

Multi-stage buil is a new feature in Docker 17.05 and allows you to optimize your Dockerfiles. With it, we can reduce our build into a single step. This is how our `Dockerfile` will look like:

```Dockerfile
FROM golang:1.9 as builder

WORKDIR /go/src/github.com/alextanhongpin/hello-world

COPY main.go .

RUN go get -d -v

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .


FROM alpine:latest  
RUN apk --no-cache add ca-certificates

WORKDIR /root/
COPY --from=builder /go/src/github.com/alextanhongpin/hello-world/app .
CMD ["./app"]

```

Let's build and observe the magic:

```bash
$ docker build -t alextanhongpin/hello-world .

Sending build context to Docker daemon  2.018MB
Step 1/10 : FROM golang:1.9 as builder
 ---> 5e2f23f821ca
Step 2/10 : WORKDIR /go/src/github.com/alextanhongpin/hello-world
 ---> Using cache
 ---> d36bf8436458
Step 3/10 : COPY main.go .
 ---> Using cache
 ---> 2fa05dc652bc
Step 4/10 : RUN go get -d -v
 ---> Using cache
 ---> bb0f73ac82d1
Step 5/10 : RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
 ---> Using cache
 ---> 8b32d3f4cfd0
Step 6/10 : FROM alpine:latest
 ---> 7328f6f8b418
Step 7/10 : RUN apk --no-cache add ca-certificates
 ---> Using cache
 ---> 70fb51eb7cf7
Step 8/10 : WORKDIR /root/
 ---> Using cache
 ---> a7a3eea586d3
Step 9/10 : COPY --from=builder /go/src/github.com/alextanhongpin/hello-world/app .
 ---> Using cache
 ---> e723f2ddc2eb
Step 10/10 : CMD ./app
 ---> Using cache
 ---> 71995c167901
Successfully built 71995c167901
Successfully tagged alextanhongpin/hello-world:latest
```

```bash
$ docker run alextanhongpin/hello-world
Hello, go!
```

You can now build your golang image in a single step. The output is shown below:

```bash
$ docker image list | grep hello-world

alextanhongpin/hello-world-01                   latest              d678076674fa        4 minutes ago       6.55MB
alextanhongpin/hello-world-00                   latest              2669fc5303bf        8 minutes ago       729MB
alextanhongpin/hello-world                      latest              71995c167901        12 hours ago        6.54MB
```


## Adding Label Schema

There is a standardised naming conventions for labels in Docker image, which can be found [here](http://label-schema.org/rc1/). In our Dockerfile, we specify the `ARG` to be passed during the build process.

```Dockerfile
FROM golang:1.9 as builder

WORKDIR /go/src/github.com/alextanhongpin/hello-world

COPY main.go .

RUN go get -d -v

RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .


FROM alpine:latest  
RUN apk --no-cache add ca-certificates

WORKDIR /root/
COPY --from=builder /go/src/github.com/alextanhongpin/hello-world/app .

# Metadata params
ARG VERSION
ARG BUILD_DATE
ARG VCS_URL
ARG VCS_REF
ARG NAME
ARG VENDOR

# Metadata
LABEL org.label-schema.build-date=$BUILD_DATE \
      org.label-schema.name=$NAME \
      org.label-schema.description="Example of multi-stage docker build" \
      org.label-schema.url="https://example.com" \
      org.label-schema.vcs-url=https://github.com/alextanhongpin/$VCS_URL \
      org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.vendor=$VENDOR \
      org.label-schema.version=$VERSION \
      org.label-schema.docker.schema-version="1.0" \
      org.label-schema.docker.cmd="docker run -d alextanhongpin/hello-world"

CMD ["./app"]
```

We create a simple Makefile to ease storing the variables:

```
VERSION := $(shell git rev-parse HEAD)
BUILD_DATE := $(shell date -R)
VCS_URL := $(shell basename `git rev-parse --show-toplevel`)
VCS_REF := $(shell git log -1 --pretty=%h)
NAME := $(shell basename `git rev-parse --show-toplevel`)
VENDOR := $(shell whoami)

print:
	@echo VERSION=${VERSION} 
	@echo BUILD_DATE=${BUILD_DATE}
	@echo VCS_URL=${VCS_URL}
	@echo VCS_REF=${VCS_REF}
	@echo NAME=${NAME}
	@echo VENDOR=${VENDOR}

build:
	docker build -t alextanhongpin/hello-go --build-arg VERSION="${VERSION}" \
	--build-arg BUILD_DATE="${BUILD_DATE}" \
	--build-arg VCS_URL="${VCS_URL}" \
	--build-arg VCS_REF="${VCS_REF}" \
	--build-arg NAME="${NAME}" \
	--build-arg VENDOR="${VENDOR}" .
```

Running `$ make print` output:

```
VERSION=a8dd38b765470fe69ee1127519a586512942f318
BUILD_DATE=Tue, 27 Mar 2018 11:50:42 +0800
VCS_URL=go-docker-multi-stage-build
VCS_REF=a8dd38b
NAME=go-docker-multi-stage-build
VENDOR=alextan
```

Running `$ make build` will now inject those variables into the Docker image during the build process:

```
docker build -t alextanhongpin/hello-go --build-arg VERSION="a8dd38b765470fe69ee1127519a586512942f318" \
	--build-arg BUILD_DATE="Tue, 27 Mar 2018 11:51:16 +0800" \
	--build-arg VCS_URL="go-docker-multi-stage-build" \
	--build-arg VCS_REF="a8dd38b" \
	--build-arg NAME="go-docker-multi-stage-build" \
	--build-arg VENDOR="alextan" .
Sending build context to Docker daemon  96.77kB
Step 1/17 : FROM golang:1.9 as builder
 ---> a6c306bd0b2f
Step 2/17 : WORKDIR /go/src/github.com/alextanhongpin/hello-world
 ---> Using cache
 ---> 13d8a2ac5144
Step 3/17 : COPY main.go .
 ---> Using cache
 ---> 3db9ab323851
Step 4/17 : RUN go get -d -v
 ---> Using cache
 ---> 1c4a3363c51c
Step 5/17 : RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .
 ---> Using cache
 ---> 16c60c3ee194
Step 6/17 : FROM alpine:latest
 ---> 3fd9065eaf02
Step 7/17 : RUN apk --no-cache add ca-certificates
 ---> Using cache
 ---> 09eef72b03f8
Step 8/17 : WORKDIR /root/
 ---> Using cache
 ---> caaa69a4ea86
Step 9/17 : COPY --from=builder /go/src/github.com/alextanhongpin/hello-world/app .
 ---> Using cache
 ---> 4f152587c422
Step 10/17 : ARG VERSION
 ---> Using cache
 ---> 238fd64c8894
Step 11/17 : ARG BUILD_DATE
 ---> Using cache
 ---> d6e82c21c2b7
Step 12/17 : ARG VCS_URL
 ---> Using cache
 ---> 4483ad9a0ebc
Step 13/17 : ARG VCS_REF
 ---> Using cache
 ---> ca3fa7d5de18
Step 14/17 : ARG NAME
 ---> Using cache
 ---> ad4d30434177
Step 15/17 : ARG VENDOR
 ---> Using cache
 ---> b5720f32c236
Step 16/17 : LABEL org.label-schema.build-date=$BUILD_DATE       org.label-schema.name=$NAME       org.label-schema.description="Example of multi-stage docker build"       org.label-schema.url="https://example.com"       org.label-schema.vcs-url=https://github.com/alextanhongpin/$VCS_URL       org.label-schema.vcs-ref=$VCS_REF       org.label-schema.vendor=$VENDOR       org.label-schema.version=$VERSION       org.label-schema.docker.schema-version="1.0"       org.label-schema.docker.cmd="docker run -d alextanhongpin/hello-world"
 ---> Running in e4aad0b65c5f
Removing intermediate container e4aad0b65c5f
 ---> fbca9a419ee3
Step 17/17 : CMD ["./app"]
 ---> Running in 450ea5afb993
Removing intermediate container 450ea5afb993
 ---> 0f8e678167f6
Successfully built 0f8e678167f6
Successfully tagged alextanhongpin/hello-go:latest
```

To verify the labels are injected into the image, you can just `docker inspect` the image.

```bash
# Inspect the labels by iterating through them, and printing them each in a new line
$ docker inspect --format='{{range $k, $v := .Config.Labels}}{{$k}}={{$v}}{{println}}{{end}}' alextanhongpin/hello-go
```

Output:

```
org.label-schema.build-date=Tue, 27 Mar 2018 11:51:16 +0800
org.label-schema.description=Example of multi-stage docker build
org.label-schema.docker.cmd=docker run -d alextanhongpin/hello-world
org.label-schema.docker.schema-version=1.0
org.label-schema.name=go-docker-multi-stage-build
org.label-schema.url=https://example.com
org.label-schema.vcs-ref=a8dd38b
org.label-schema.vcs-url=https://github.com/alextanhongpin/go-docker-multi-stage-build
org.label-schema.vendor=alextan
org.label-schema.version=a8dd38b765470fe69ee1127519a586512942f318
```

<!--
Feedback from Chee Leong:

I think for the go app, you should put the executable to `/bin/` `/usr/bin` or `/usr/local/bin` so it’ll be not WORKDIR reliant.


[9:31] 
for the `apk` command, if the index is outdated, you might face error running that.
-->
