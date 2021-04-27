---
layout: post
title: "Golang app development using Skaffold"
category: 
tags: []
---

# Introduction

Developing an application to be distributed as a K8s service is a
complicated undertaking. Besides learning the application language and
solving the application problem, there are all the K8s workflows that
need to be automated. This is my attempt to navigate the insane K8s
ecosystem of tools as I try to make a decent development and
production workflow.

# Development workflow

The local development workflow needs to have a fast feedback
loop. For a K8s application that requires at minimum a container build
and deployment.

# Ubuntu setup of go 1.15

Latest go at this time is 1.16.3, but for a sample app the distro
supplied 1.15 is fine.

    sudo apt install golang-1.15
    cd $HOME/bin && ln -s /usr/lib/go-1.15/bin/go

I have $HOME/bin in my $PATH so this makes it easy to manage all the
installation of single binary tools. Technically with buildpacks I
don't even need to install the go toolchain, but I want to explore
things like debugging of a running go application.

# Buildpacks

I am not a big fan of Dockerfiles, especially the multi-stage
Dockerfiles which is the right way to separate the build and runtime
containers. Due to the single binary structure of Go applications they
can have a tiny runtime image. So I decided to investigate using
buildpacks[4] which look like a much better alternative for application
development. It even supports new features like reproducible builds
and image rebasing.

Install the pack tool.

    cd $HOME/bin
    curl -LO https://github.com/buildpacks/pack/releases/download/v0.18.1/pack-v0.18.1-linux.tgz
    tar xzf pack-v0.18.1-linux.tgz
    chmod +x pack
    rm -f pack-v0.18.1-linux.tgz

Start with golang buildpacks sample app[1].

Since this is a golang app, the default buildpack can be tiny:

    pack config default-builder paketobuildpacks/builder:tiny
    cd $APP
    pack build mod-sample --buildpack gcr.io/paketo-buildpacks/go

With buildpacks running locally, the workflow is <edit> and save, run
pack to build, run docker and test. It takes a few seconds and
multiple steps.

# Skaffold and Minikube

This app will run in K8s and will depend on K8s features so it will
need to run inside K8s. Enter Minikube[2] for a local K8s setup and
Skaffold[3] to orchestrate the development workflow.

    cd $HOME/bin
    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    chmod +x minikube
    minikube start

This starts up a full K8s instance locally using the docker
driver. The initial download was ~1GB so it takes a while.

    cd $HOME/bin
    curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
    chmod +x skaffold

Now switch to the golang buildpack sample with skaffold[5].

    <term 1> skaffold dev
    <term 2> minikube tunnel
    <term 3> kubectl get svc # to get IP
    <term 3> curl -s http://<external IP>:8080

minikube has its own docker daemon and the buildpacks used and images
built are located inside minikube and not the host docker[6].

    <term 4> eval $(minikube docker-env)
    <term 4> docker images

This makes deploying the image very fast because it isn't copied.

# Development workflow

The skaffold sample is setup to use `gcr.io/buildpacks/builder:v1` and
it also works with the builder `paretobuildpacks/builder:tiny`. The
Google buildpacks[7] supports "file sync" which copies changed files
directly to the image. This means changes are available in seconds
which is great for development.

# Next steps

My application will be a multi-cluster app that exchanges K8s resource
data. First step is to query the resource utilization of the K8s
cluster using client-go.

[1]: https://github.com/paketo-buildpacks/samples/tree/main/go/mod
[2]: https://minikube.sigs.k8s.io
[3]: https://skaffold.dev
[4]: https://buildpacks.io
[5]: https://github.com/GoogleContainerTools/skaffold/tree/master/examples/buildpacks
[6]: https://minikube.sigs.k8s.io/docs/handbook/pushing/
[7]: https://github.com/GoogleCloudPlatform/buildpacks
