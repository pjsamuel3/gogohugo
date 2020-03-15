+++
date = "2020-01-10T16:43:14+01:00"
draft = false
weight = 220
description = "DESC"
title = "My post"
bref="Kube has a basic set of colors for text, links, buttons, states and gray palette. These colors help to create uniformity and harmony in the look of UI elements. All colors are carefully selected and combined with each other. Of course, you can change the color scheme to your choice in the framework settings."
toc = true
+++

---
title: Go repos within Amedia
---

There are really no strict rules within Amedia on how to organize a Go project. But
in order to make use of our tools for CI and deployment, it has to adhere to some
basic conventions.

# Preparing a Go app for release and deployment

As all other services within Amedia, a Go service will be released as a Docker image and
deployed to Kubernetes somewhere.

It is critical that the released code are compiled, tested and build in an environment identical to the Docker image deployed to production. The best way to achieve this is by running the entire build process within Docker. But instead of having different Dockerfile's for build and release, we recommend using [Docker Multi-Stage builds](https://docs.docker.com/develop/develop-images/multistage-build/).

This way we keep everything within one *Dockerfile* and provides a consistent
build process acrosse machines and environments.

## Creating a Dockerfile for your project

Exactly how your Dockerfile should end up depends on the folder structure, tools and so on. But the main structure should more or less be the same. Let's look at a real-life example from Iscado (with comments):

```docker
# ----------------------------------------------------
# Deps stage
#
# Make sure all dependencies are checked and downloaded
# ----------------------------------------------------
# By using our go-app docker image you get access to
# the private go packages at https://github.com/amedia
# Find latest release here:
# https://github.com/amedia/dokka/tree/master/amedia/go-app
FROM dr.api.no/amedia/go-app:1.10.3-3 AS deps
# Modify to match your project name
ADD . /go/src/github.com/amedia/iscado
WORKDIR /go/src/github.com/amedia/iscado
RUN dep ensure

# ----------------------------------------------------
# Test stage
#
# Run tests as soon as possible.
# ----------------------------------------------------
FROM deps AS test
RUN go test -v ./cmd/... ./pkg/...

# ----------------------------------------------------
# Build stage
#
# Build the binaries
# ----------------------------------------------------
FROM deps AS build
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -a -installsuffix cgo -o /app/bin/iscado ./cmd/iscado

# ----------------------------------------------------
# Release stage
# ----------------------------------------------------
# Some people prefer running Go apps in Scratch, but
# having a shell is nice too :)
FROM alpine:3.6 as release
ENV LC_ALL=en_US.UTF-8
ENV LC_LANG=en_US.UTF-8
ENV LC_LANGUAGE=en_US.UTF-8
# Copy binaries from the build stage into this image
COPY --from=build /app/bin/ /app/bin
# Add extra files needed from your project
ADD etc/ca-certificates.crt /etc/ssl/certs/
ADD config/pubsub-jwt.json /app/config/
WORKDIR /app
ENTRYPOINT ["/app/bin/iscado"]
EXPOSE 9802
```

`docker build .` will build the binary, run the tests and build the final Docker image. Be aware that You need Docker 17.05+ for Multi-Stage builds to work.

## COP setup

[COP](devops/release.html) is our tool for releasing apps and libraries. Creating a `cop.json` for your project is pretty simlpe. Again, let's look
at an example from Iscado:

```json
{
  "cop.version": 1.0,
  "builds": [
    {
      "build": {
        "name": "iscado",
        "type": "docker",
        "tool": "semver",
        "access": "private"
      },
      "release": {
        "docker": {
          "target": [
            "eu.gcr.io/amedia-analytics-eu"
          ]
        }
      }
    }
  ]
}
```

For docker builds the trick is to set `tool` to `semver`. And as you might have guessed already, you will need a `.semver` file in your project. Initially it should look something like this.:

```text
---
:major: 0
:minor: 0
:patch: 1
:special: ''
```

# Note about private Go packages

First of all, go-mod will use http to download private repos, just as it does with public ones. Since we use ssh to access github, this wont work. so we have to add this config to our local git client:
`git config --global url."git@github.com:".insteadOf "https://github.com/"`

In order for Docker to get access to our private Go packages during builds, each of these GitHub repositories must give the `go-dependencies` team read access.

![Permission example](/images/handbook/golang/access.png)

Note: You need to be GitHub admin to do this.
