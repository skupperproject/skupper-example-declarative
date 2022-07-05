# Skupper Declarative Hello World

[![main](https://github.com/skupperproject/skupper-example-hello-world/actions/workflows/main.yaml/badge.svg)](https://github.com/skupperproject/skupper-example-hello-world/actions/workflows/main.yaml)

#### A minimal HTTP application deployed across Kubernetes clusters using Skupper


This example is part of a [suite of examples][examples] showing the
different ways you can use [Skupper][website] to connect services
across cloud providers, data centers, and edge sites.

[website]: https://skupper.io/
[examples]: https://skupper.io/examples/index.html


#### Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Step 1: Configure separate console sessions](#step-1-configure-separate-console-sessions)
* [Step 2: Access your clusters](#step-2-access-your-clusters)
* [Step 3: Set up your namespaces](#step-3-set-up-your-namespaces)
* [Step 4: Install Skupper in your namespaces](#step-4-install-skupper-in-your-namespaces)
* [Step 5: Check the status of your namespaces](#step-5-check-the-status-of-your-namespaces)
* [Step 6: Link your namespaces](#step-6-link-your-namespaces)
* [Step 7: Deploy the frontend and backend services](#step-7-deploy-the-frontend-and-backend-services)
* [Step 8: Expose the backend service](#step-8-expose-the-backend-service)
* [Step 9: Expose the frontend service](#step-9-expose-the-frontend-service)
* [Step 10: Test the application](#step-10-test-the-application)
* [Accessing the web console](#accessing-the-web-console)
* [Cleaning up](#cleaning-up)
* [Summary](#summary)

## Overview

This example is a very simple multi-service HTTP application that can
be deployed across multiple Kubernetes clusters using Skupper 
with a declarative approach.

It contains two services:

* A backend service that exposes an `/api/hello` endpoint.  It
  returns greetings of the form `Hi, <your-name>.  I am <my-name>
  (<pod-name>)`.

* A frontend service that sends greetings to the backend and
  fetches new greetings in response.

With Skupper, you can place the backend in one cluster and the
frontend in another and maintain connectivity between the two
services without exposing the backend to the public internet.

<img src="images/entities.svg" width="640"/>

## Prerequisites


* The `kubectl` command-line tool, version 1.15 or later
  ([installation guide][install-kubectl])

* The `skupper` command-line tool, the latest version ([installation
  guide][install-skupper])

* Access to at least one Kubernetes cluster, from any provider you
  choose

[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[install-skupper]: https://skupper.io/install/index.html


## Step 1: Configure separate console sessions

Skupper is designed for use with multiple namespaces, typically on
different clusters.  The `skupper` command uses your
[kubeconfig][kubeconfig] and current context to select the
namespace where it operates.

[kubeconfig]: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

Your kubeconfig is stored in a file in your home directory.  The
`skupper` and `kubectl` commands use the `KUBECONFIG` environment
variable to locate it.

A single kubeconfig supports only one active context per user.
Since you will be using multiple contexts at once in this
exercise, you need to create distinct kubeconfigs.

Start a console session for each of your namespaces.  Set the
`KUBECONFIG` environment variable to a different path in each
session.

_**Console for west:**_

~~~ shell
export KUBECONFIG=~/.kube/config-west
~~~

_**Console for east:**_

~~~ shell
export KUBECONFIG=~/.kube/config-east
~~~

## Step 2: Access your clusters

The methods for accessing your clusters vary by Kubernetes
provider. Find the instructions for your chosen providers and use
them to authenticate and configure access for each console
session.  See the following links for more information:

* [Minikube](https://skupper.io/start/minikube.html)
* [Amazon Elastic Kubernetes Service (EKS)](https://skupper.io/start/eks.html)
* [Azure Kubernetes Service (AKS)](https://skupper.io/start/aks.html)
* [Google Kubernetes Engine (GKE)](https://skupper.io/start/gke.html)
* [IBM Kubernetes Service](https://skupper.io/start/ibmks.html)
* [OpenShift](https://skupper.io/start/openshift.html)
* [More providers](https://kubernetes.io/partners/#kcsp)

## Step 3: Set up your namespaces

Use `kubectl create namespace` to create the namespaces you wish
to use (or use existing namespaces).  Use `kubectl config
set-context` to set the current namespace for each session.

_**Console for west:**_

~~~ shell
kubectl create namespace west
kubectl config set-context --current --namespace west
~~~

_Sample output:_

~~~ console
$ kubectl create namespace west
namespace/west created

$ kubectl config set-context --current --namespace west
Context "minikube" modified.
~~~

_**Console for east:**_

~~~ shell
kubectl create namespace east
kubectl config set-context --current --namespace east
~~~

_Sample output:_

~~~ console
$ kubectl create namespace east
namespace/east created

$ kubectl config set-context --current --namespace east
Context "minikube" modified.
~~~

## Step 4: Install Skupper in your namespaces

Deploy a site controller and create a site.

_**Console for east:**_

~~~ shell
kubectl apply -f https://raw.githubusercontent.com/skupperproject/skupper/master/cmd/site-controller/deploy-watch-current-ns.yaml
kubectl apply -f ./backend/east-site.yml
~~~

_**Console for west:**_

~~~ shell
kubectl apply -f https://raw.githubusercontent.com/skupperproject/skupper/master/cmd/site-controller/deploy-watch-current-ns.yaml
kubectl apply -f ./frontend/west-site.yml
~~~

## Step 5: Check the status of your namespaces

Use `skupper status` in each console to check that Skupper is
installed.

_**Console for west:**_

~~~ shell
skupper status
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "west" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
~~~

_**Console for east:**_

~~~ shell
skupper status
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "east" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
~~~

As you move through the steps below, you can use `skupper status` at
any time to check your progress.

## Step 6: Link your namespaces

Apply yaml to create a token from west and use that token in east.

_**Console for west:**_

~~~ shell
kubectl apply -f ./frontend/token-request.yml
sleep 1s
kubectl get secret -o yaml west-link | yq 'del(.metadata.namespace)' > ~/west-link.yaml
~~~

_**Console for east:**_

~~~ shell
kubectl apply -f ~/west-link.yaml
skupper link status --wait 60
~~~

## Step 7: Deploy the frontend and backend services

Use `kubectl create deployment` to deploy the frontend service
in `west` and the backend service in `east`.

_**Console for west:**_

~~~ shell
kubectl create deployment frontend --image quay.io/skupper/hello-world-frontend
~~~

_Sample output:_

~~~ console
$ kubectl create deployment frontend --image quay.io/skupper/hello-world-frontend
deployment.apps/frontend created
~~~

_**Console for east:**_

~~~ shell
kubectl create deployment backend --image quay.io/skupper/hello-world-backend --replicas 3
~~~

_Sample output:_

~~~ console
$ kubectl create deployment backend --image quay.io/skupper/hello-world-backend --replicas 3
deployment.apps/backend created
~~~

## Step 8: Expose the backend service

We now have two namespaces linked to form a Skupper network, but
no services are exposed on it.  Skupper uses `skupper.io` 
annotations to select a service from one namespace for
exposure on all the linked namespaces.

Use annotations to expose the backend service to the
frontend service.

_**Console for east:**_

~~~ shell
kubectl annotate deployment/backend skupper.io/proxy="http"
kubectl annotate deployment/backend skupper.io/port="8080"
~~~

## Step 9: Expose the frontend service

We have established connectivity between the two namespaces and
made the backend in `east` available to the frontend in `west`.
Before we can test the application, we need external access to
the frontend.

Use `kubectl expose` with `--type LoadBalancer` to open network
access to the frontend service.

_**Console for west:**_

~~~ shell
kubectl expose deployment/frontend --port 8080 --type LoadBalancer
~~~

_Sample output:_

~~~ console
$ kubectl expose deployment/frontend --port 8080 --type LoadBalancer
service/frontend exposed
~~~

## Step 10: Test the application

Now we're ready to try it out.  Use `kubectl get service/frontend`
to look up the external IP of the frontend service.  Then use
`curl` or a similar tool to request the `/api/health` endpoint at
that address.

**Note:** The `<external-ip>` field in the following commands is a
placeholder.  The actual value is an IP address.

_**Console for west:**_

~~~ shell
kubectl get service/frontend
curl http://<external-ip>:8080/api/health
~~~

_Sample output:_

~~~ console
$ kubectl get service/frontend
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
frontend   LoadBalancer   10.103.232.28   <external-ip>   8080:30407/TCP   15s

$ curl http://<external-ip>:8080/api/health
OK
~~~

If everything is in order, you can now access the web interface by
navigating to `http://<external-ip>:8080/` in your browser.

## Accessing the web console

Skupper includes a web console you can use to view the application
network.  To access it, use `skupper status` to look up the URL of
the web console.  Then use `kubectl get
secret/skupper-console-users` to look up the console admin
password.

**Note:** The `<console-url>` and `<password>` fields in the
following output are placeholders.  The actual values are specific
to your environment.

_**Console for west:**_

~~~ shell
skupper status
kubectl get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "west" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'

$ kubectl get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
<password>
~~~

Navigate to `<console-url>` in your browser.  When prompted, log
in as user `admin` and enter the password.

## Cleaning up

To remove Skupper and the other resources from this exercise, use
the following commands.

_**Console for west:**_

~~~ shell
skupper delete
kubectl delete service/frontend
kubectl delete deployment/frontend
~~~

_**Console for east:**_

~~~ shell
skupper delete
kubectl delete deployment/backend
~~~

## Summary

This example locates the frontend and backend services in different
namespaces, on different clusters.  Ordinarily, this means that they
have no way to communicate unless they are exposed to the public
internet.

Introducing Skupper into each namespace allows us to create a virtual
application network that can connect services in different clusters.
Any service exposed on the application network is represented as a
local service in all of the linked namespaces.

The backend service is located in `east`, but the frontend service
in `west` can "see" it as if it were local.  When the frontend
sends a request to the backend, Skupper forwards the request to the
namespace where the backend is running and routes the response back to
the frontend.

<img src="images/sequence.svg" width="640"/>

## Next steps


Check out the other [examples][examples] on the Skupper website.
