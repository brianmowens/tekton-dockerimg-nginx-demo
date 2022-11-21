# tekton-dockerimg-nginx-demo
Demonstrate using Tekton to build and publish an Nginx docker container. Helm chart included for use with ArgoCD.

This repository, while public, mostly serves as notes for me to recreate my environments at a later date.

A helpful guide for setting up Tekton triggers: 
https://www.arthurkoziel.com/tutorial-tekton-triggers-with-github-integration/#requirements

GitHub webhook payloads:
https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#webhook-payload-object-common-properties

## My Environment

A single microk8s node running in Oracle Virtual Box on a Windows laptop. Virtual box VM is created by vagrant and configured via ansible. Microk8s is configured to have ingress setup as well as metallb. Metallb is configured to use an IP range overlapping my home network (192.168.1.x-192.168.1.x)

## Overview

This demo builds a basic Nginx docker container and pushes it to a docker registry. The Nginx container will have a custom index.html placed within it during the build. This custom html page not only allows for you to easily trigger the build by adding a new message but also allows you to ensure the build workflow completed by letting you view the message on the built and deployed container.

This demo also uses a namespace called `tekton-demo`. This is hardcoded in some of the RBAC files, so it needs to be updated if using another namespace. I did this for my environment because I dislike having anything get lumped into a default namespace.

A debug task is currently in the pipeline between the git clone and kaniko build steps. The only purpose of this debug task is to dump the contents of the workspace to ensure your workspace is being properly shared between tasks and not deleted after a task completes.

## Configuring

### Create namespace

```
kubectl create namespace tekton-demo
```

### Secrets

Two secrets will be needed. One to read from GitHub in the case of a private repository and another to push to dockerhub. Your GitHub credential should leverage a Personal Access Token.

GitHub
```
kubectl create secret generic GITHUB-SECRET --from-literal=username=myuser --from-literal=password=PAT_TOKEN -n tekton-demo
```

DockerHub
```
kubectl create secret docker-registry DOCKER-REGISTRY --docker-username=myuser --docker-password=mypassword --docker-email=myemail@domain.com -n tekton-demo
```

### Tasks and Pipeline

Tasks
```
kubectl apply -f tekton-tasks/task-git-clone.yml -n tekton-demo
kubectl apply -f tekton-tasks/task-busybox-debug.yml -n tekton-demo
kubectl apply -f tekton-tasks/task-kaniko-build.yml -n tekton-demo
```
Pipeline
```
kubectl apply -f tekton-pipeline/pipeline-build-nginx-demo.yml -n tekton-demo
```

### Triggers

The secrets file should be edited to include your own secret. This secret will be shared with the GitHub webhook.

Triggers
```
kubectl apply -f tekton-triggers/rbac.yml -n tekton-demo
kubectl apply -f tekton-triggers/triggerbinding.yml -n tekton-demo
kubectl apply -f tekton-triggers/triggertemplate.yml -n tekton-demo
kubectl apply -f tekton-triggers/secret.yml
kubectl apply -f tekton-triggers/eventlistener.yml -n tekton-demo
kubectl apply -f tekton-triggers/ingress.yml -n tekton-demo
```

### Hosts File

All ingress files in this repo uses *.example.com addresses. You should update your local hosts file to point to the LoadBalancer IP of your ingress service for each ingress route.

```hosts
192.168.1.150 tektontrigger.example.com
```

## CLI Usage

If running a pipeline from the Tekton CLI, a sample command is found below. It should be updated to use the correct name of your GitHub and DockerHub secrets as well as using the supplied `tekton-pipeline/volumeClaimTemplate.yml` file to create your workspace.

```
tkn pipeline start nginx-build-demo -n tekton-demo -w name=docker-workspace,volumeClaimTemplateFile=volumeClaimTemplate.yml -p git-credential-secret-name=MY-GITHUB-SECRET -p git-url=https://github.com/brianmowens/tekton-dockerimg-nginx-demo.git -p git-branch=develop -p docker-registry=brianowens/nginx-hello-world -p docker-registry-secret-name=MY-DOCKERHUB-SECRET -p dockerfile-path=docker/Dockerfile -p image-tag=latest -p kaniko-image-url=gcr.io/kaniko-project/executor:v1.9.1
```

## Ngrok

Ngrok is a useful tool if you can expose your local Kubernetes cluster on your home or private network. If using a cloud provider, this doesn't really matter. Ngrok will allow provide you with a public URL that resolves to a local network address. 

Run Ngrok, then use the supplied URL to configure the Webhook on your git repository. Update the URLs to match what's inside your `tekton-triggers/ingress.yml` file.

```
ngrok http --host-header=tektontrigger.example.com http://tektontrigger.example.com
```