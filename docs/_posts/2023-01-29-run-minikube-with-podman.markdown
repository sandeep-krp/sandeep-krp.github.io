---
title:  "Experiment with Kubernetes on your local machine(Mac)"
date:   2023-01-29 18:00:00 +0530
categories: tech kubernetes minikube podman
---

## TL;DR
- `brew install minikube@v1.29.0`
- `brew install podman@4.3.1`
- `podman machine init --cpus 2 --memory 8288 --disk-size 50`
- `podman machine start`
- `minikube start --driver=podman --container-runtime=containerd --kubernetes-version=v1.23.12`
- `alias kubectl='minikube kubectl'`
- `kubectl get ns`


## Description
If you are staring you journey of Kubernetes, you need a Kubernetes server to play with. Creating a Kubernetes server on your cloud provider (like AWS, GCE, etc) would be ideal but for experimentation you might not have that privilege. 
[Docker desktop](https://www.docker.com/products/docker-desktop/) is a good option if you are planning to run it in your personal laptop because the company Docker does not allow you to use it in an organization without a license.
The good news is we have a free software minikube using which we can start a Kubernetes server on your local machine and do all your experiments without worrying about any licensing. Meaning, this will work for both your personal and office use.

## Install minikube
Hopefully you have [brew](https://brew.sh) installed and if not, do install it and then run:\
`brew install minikube@v1.29.0`\
and your Mac will install minikube for you. But if you follow the [official guide](https://minikube.sigs.k8s.io/docs/start/) to install it then you will see that it throws:
```
Exiting due to DRV_DOCKER_NOT_RUNNING: Found docker, but the docker service isn't running. Try restarting the docker service.
```
The reason is absence of what is called a runtime for container. Installing minkube does not install this runtime and it has to be installed separately. We have to choose from multiple runtime and the best one to use without worrying about any licensing is [podman](https://podman.io).
## Install podman
Just run:\
`brew install podman@4.3.1`\
, and it will install the latest version of podman for you. It is also beneficial to understand a little bit about how podman works. What is basically does is allows you to create a new virtual machine on your laptop/desktop. This VM is assigned certain memory and CPU that will be used to run container.

## Start the podman virtual machine
This should also be easy. Just run:\
`podman machine init --cpus 2 --memory 8288 --disk-size 50`\
You can tweak the parameters based on your machine configurations. After initialization, you have start the VM by running:\
`podman machine start`\
If you face any difficulties, you can run `podman machine rm` to delete the VM and try to start it again with a little googling.

## Start minikube (Kubernetes) server
Finally, we start the main kubernetes server:\
`minikube start --driver=podman --container-runtime=containerd --kubernetes-version=v1.23.12`\
You can try running it without `--kubernetes-version=v1.23.12` but it sometimes does not work because then minikube tries to start the latest server version. It is better that you set it to a version which is also your target version in your office.

## Use minikube kubectl as your client
This should also be easy:\
`alias kubectl='minikube kubectl'`\
Once this is done, you should be able to do `kubectl get ns`. If you see a list of namespaces, you can now start experimenting with your local kubernetes cluster.
