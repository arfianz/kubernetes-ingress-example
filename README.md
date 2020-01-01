# Kubernetes Ingress Example App

Using minikube to run this project. Although, minikube supports (almost) everything you would expect from a kubernetes cluster but since it’s running locally, certain cloud provider features will not work out of the box. One such feature is Ingress.

## What's Ingress?

An ingress is a set of rules that allows inbound connections to reach the kubernetes cluster services.

Typically, the kubernetes cluster is firewalled from the internet. It has edge routers enforcing the firewall. Kubernetes resources like services, pods, have IPs only routable by the cluster network and are not (directly) accessible outside the cluster. All traffic that ends up at an edge router is either dropped or forwarded elsewhere. So, submitting an ingress to the cluster defines a set of rules for routing external traffic to the kubernetes endpoints.

As you might have guessed, the rule is:

- all requests to myminikube.info/ should be routed to the service in the cluster named echoserver.
- requests mapping to cheeses.all/stilton should be routed to the stilton-cheese service.
- and finally, requests mapping to cheeses.all/cheddar should be routed to the cheddar-cheese service.

Of course, there’s more to it; like the backend tag which implies that unmatched requests should be routed to the default-http-backend service and there’s also the familiar kubernetes tags; for example the apiVersion tag which clearly marks ingress as a beta feature.

Note the annotation:

```bash
ingress.kubernetes.io/rewrite-target: /
```

This is necessary if the target services expect requests from the root URL i.e cheeses.all and not cheeses.all/stilton . The ingress mapping by default will pass along the trailing path on to the service (e.g /stilton) and if the service doesn’t accept request on that path you get a 403 error response. Thus with rewrite-target annotation, the request path is rewritten with the given path before the request get’s forwarded to the target backend.

## The ingress controller

In order for the Ingress resource to work, the cluster must have an Ingress controller running. When a user requests an ingress by POSTing an Ingress resource (such as the one above) to the API server, the Ingress controller is responsible for fulfilling the Ingress, usually with a loadbalancer. Though it may also configure the edge routers or additional frontends to help handle the traffic.

As such without an Ingress controller to satisfy the ingress, merely creating the ingress resource will have no effect.

You can write your own controller but you need not to. There are readily available third-party ingress controllers like the Nginx, Traefik, HAproxy controllers which you could easily leverage. I will be using the Nginx controller for this demo but feel free to try out any other.

## Setup

Minikube v1.6.2 (and above) ships with Nginx ingress setup as an add-on (as requested here). It can be easily enabled by executing.

```bash
minikube addons enable ingress
```

Enabling the add-on provisions the following:
- a configMap for the Nginx loadbalancer
- the Nginx ingress controller
- a service that exposes a default Nginx backend pod for handling unmapped requests.

If you are using an older version of minikube (and insist on not updating) you might need to manually deploy the ingress controller (and default backend service).

The layout of our cluster for this demo is:

- A backend that will receive requests for myminikube.info and displays some basic information about the cluster and the request.
- A pair of backends that will receive the request for cheeses.all .One whose path begins with /stilton and another whose path begins with /cheddar .

Nginx already provides a default backend so we need not worry about that.

Let’s set up the echoserver deployments and expose them:

```bash
kubectl run echoserver --image=gcr.io/google_containers/echoserver:1.4 --port=8080
kubectl expose deployment echoserver --type=NodePort
```

Then confirm that requests can get to the service

```bash
minikube service echoserver
```

This should open the service in your default browser.

Next we setup the backend for /stilton cheese

```bash
kubectl run stilton-cheese --image=errm/cheese:stilton --port=80
kubectl expose deployment stilton-cheese --type=NodePort
```

You can also check out the service:

```bash
minikube service stilton-cheese
```

Finally the /cheddar cheese

```bash
kubectl run cheddar-cheese --image=errm/cheese:cheddar --port=80
kubectl expose deployment cheddar-cheese --type=NodePort
minikube service cheddar-cheese
```

Thus far, we can access the services via the [minikube ip]:[node port] address. Our aim, however, is to access them via myminikube.info , cheeses.all/stilton and cheeses.all/cheddar . And that’s where ingress comes in.

To setup ingress, enable the minikube add-on

```bash
minikube addons enable ingress
```

Copy the ingress definition above and save to a file ingress-demo.yaml Then we create the ingress resource

```bash
kubectl create -f ingress-demo.yaml
```

You can run kubectl describe ing ingress-tutorial for information on the requested ingress.

Now, the last bit is to update our /etc/hosts file to route requests from myminikube.info and cheeses.all to our minikube instance.

Execute

```bash
echo "$(minikube ip) myminikube.info cheeses.all" | sudo tee -a /etc/hosts
```

to add the following lines to your /etc/hosts file.

```bash
[minikube ip] myminikube.info cheeses.all
```

[minikube ip] will be replaced with the actual IP of your minikube instance.

And our work is done.

Test it out by visiting myminikube.info , cheeses.all/stilton , cheeses.all/cheddar from your browser

## Basic Debugging

In this section, we will explore a few basics of debugging. The following show how to gather basic information, which can be useful to determine what’s going on.

### Describe the Pods

Prints a detailed description of the selected pods, which includes events.

```bash
kubectl get pods -n kube-system | grep nginx-ingress-controller
kubectl describe pods -n kube-system nginx-ingress-controller-xxxxxx-yyyy
```

### View the Logs

Prints the logs for the nginx-ingress-controller.

```bash
kubectl logs -n kube-system nginx-ingress-controller-xxxxxx-yyyy
```

### View the Nginx Conf

Displays how nginx configures the application routing rules.

```bash
kubectl exec -it -n kube-system nginx-ingress-controller-xxxxxx-yyyy cat /etc/nginx/nginx.conf
```

## LICENSE

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

