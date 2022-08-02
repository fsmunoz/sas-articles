---
title: SAS Viya, Kubernetes and networking: a gentle introduction
author: Frederico Muñoz <frederio.munoz@sas.com>
---

# Introdution


> In essence, Kubernetes is the sum of all the bash scripts and best practices
> that most system administrators would cobble together over time, presented as a
> single system behind a declarative set of APIs.
>
> -- <cite>Kelsey Hightower</cite>

SAS Viya has adopted Kubernetes as the underlying container orchestration
platform, opening up immense possibilities for all that wish to combine the
industry-leading abilities of SAS in analytics, data science and AI, with a
cloud-native deployment approach that aligns with current strategic roadmaps,
not the least because of the potential benefits it brings to the table:
integration with CI/CD for deployment and code, enabling hybrid cloud scenarios
by supporting both on-premises and cloud deployments, multi-cloud support by
default, and perhaps more importantly, a product development process that is
more agile, with more releases, a faster incorporation of new features.

![Evolution of deployment strategies](https://d33wubrfki0l68.cloudfront.net/26a177ede4d7b032362289c6fccd448fc4a91174/eb693/images/docs/container_evolution.svg)

If the move towards Kubernetes seems obvious today (especially considering that
Kubernetes has become the industry standard in container orchestration, and is
the underlying platform on which a lot of other extremely popular technologies
are built on), someone coming from a SAS 9 background can find the target
architecture somewhat hermetic: instead of a number of servers with a specific
set of well-known components, SAS Viya depends on nodes, deployments, services,
ingresses, and pods, plus many other concepts that can seem complex - and
especially so in terms of the network aspects.

![SAS Studio in SAS Viya](./sas-studio-window.png)

When using SAS Studio in SAS Viya, how do our requests get there? This is
precisely the aspect we will address in this article: how do network requests
work in a Kubernetes context, starting from a single pod and working our way out
of the Kubernetes onion, peeling off the apparent complexity layer by layer -
and once we are comfortably on the outside, trace our way back, now with a
clearer understanding of how SAS Viya and Kubernetes work.

# Kubernetes: an overview of the network model

There is a lot of good information about Kubernetes networking, starting from
[the official
one](https://kubernetes.io/docs/concepts/cluster-administration/networking/). In
more than one way, the Kubernetes network model is a set of principles whose
implementation is done by specific components (namely, Container Network
Plugins, CNI).

It's not our intention to go into the details of all the options, and the
possibilities of using different approaches. Our environment will be based on

* A Kubernetes 1.23 cluster on Microsoft Azure (Azure Kubernetes Service).
* SAS Viya 2022.1.3, deployed in a `sas-viya` namespace (configured to be the
  active Kubernetes context)
* Use of the AKS default `kubenet` CNI.
* Use of the NGINX Ingress controller

![SAS Viya on Azure Kubernetes Service](https://documentation.sas.com/api/docsets/itopscon/v_015/content/images/aks-arch.svg?locale=en)

Given that we are describing standard Kubernetes concepts, the vast majority of
this article will apply to all Kubernetes and SAS Viya versions.

[Kubenet](https://docs.microsoft.com/en-us/azure/aks/concepts-network#kubenet-basic-networking)
is specific to Microsoft Azure, and the network model can be s

1. Nodes receive an IP address from the Azure virtual network subnet.
2. Pods receive an IP address from a logically different address space than the
   nodes' Azure virtual network subnet.
3. Network address translation (NAT) is then configured so that the pods can
   reach resources on the Azure virtual network.
4. The source IP address of the traffic is translated to the node's primary IP
   address.

We will refrain from explaining some of the implications of these points: as we
go along our step-by-step exploration of how a typical HTTP(S) request, they
will become more obvious.

This diagram is a good starting point for the concepts we will explore:

![SAS Viya on AKS](./aks-sas-overview.png)


* SAS Viya, as a solution, is composed of Pods, Services and Ingresses, just a
few of the many other types of components (including Deployments, container
images, Secrets, etc.): those were chosen given the very direct role they play
when describing the network flows.
* The NGINX Ingress controller is a Layer 7 proxy and HTTP server, a requirement
for SAS Viya due to the way it allows URL-based routing, among other features.
* The entire solution is deployed in Microsoft Azure, namely the Azure
Kubernetes Service and the Azure Load Balancer.


# From the inside out

For our journey, we will use SAS Studio as our starting point: while other
components in SAS Viya will have different configurations, SAS Studio is a good
example of how most microservices in SAS Viya work.

The following image shows the network configuration of the cluster; there's no
need to fully understand what this means right now, but we will be referring
back to this information as we go along.

![AKS network overview](./aks_network_overview.png)

We should note that the details above match our earlier description of the
`kubenet` CNI, and in simple terms determines which address space is reserved
for specific uses in the cluster.

With that out of the way, let's start!

## Microservices: the Pod

![The SAS Studio pod](./pod-view.png)

The actual application that will interact with our requests - the closest
equivalent to what would be the application program in a "traditional" server
installation - is running inside a pod, [the smallest deployable units of
computing deployed in
Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/),a group of one
or more containers that share storage and network resources, and that are always
co-scheduled.

SAS Viya is composed of many pods, and we can easily get the one that has the
SAS Studio application running:

```console
$ k get pod|grep sas-studio-app
sas-studio-app-75b59846b4-2s69z                                   1/1     Running      0             37m
```

Taking a look into that pod, we can see that it's running very few processes,
all of which are directly related with the application: unlike a typical Linux
server, there aren't lots of background processes and other applications
running. This is one of the advantages of containers: we have a light-weight
Linux server dedicated to SAS Studio, complete with all the necessary
dependencies.


```console
$ k  exec --stdin --tty sas-studio-app-75b59846b4-2s69z -- ps -efw
UID        PID  PPID  C STIME TTY          TIME CMD
sas          1     0  0 Jul31 ?        00:00:01 /usr/local/bin/tini -- /opt/sas/viya/home/bin/sas-studio-app
sas          7     1  0 Jul31 ?        00:00:00 /bin/bash /opt/sas/viya/home/bin/sas-studio-app
sas         50     7  0 Jul31 ?        00:04:26 /usr/lib/jvm/java-11-openjdk-11.0.15.0.9-2.el8_5.x86_64/bin/java --add-opens java.base/java.lang=ALL-UNNAMED -D
sas       1877     0  3 09:04 pts/0    00:00:00 ps -efw
```

We can also infer that SAS Studio is a Java application: microservices make
so-called "polyglot" applications trivial to develop, and SAS Viya is one
example of that (apart from Java, there are microservices written in Go, for
example).

Back into our SAS Studio application, we can get the IP that it's listening to
easily enough:

```console
$ k get pod sas-studio-app-75b59846b4-2s69z -o wide
NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE                               NOMINATED NODE   READINESS GATES
sas-studio-app-75b59846b4-2s69z   1/1     Running   0          14h   10.244.2.21   aks-stateful-39930745-vmss000005   <none>           <none>

```

The IP, `10.244.2.21` is within the Pod CIDR that we seen in the `kubenet`
configuration summary: all pods get an IP from this network segment, and by
default they can be contacted using that IP and the right port. In our example,
the ports where SAS Studio listens to are 8080 and 10445:

```console
$ k get pod sas-studio-app-75b59846b4-2s69z  -o jsonpath='{.spec.containers[*].ports}'
[{"containerPort":8080,"name":"http","protocol":"TCP"},{"containerPort":10445,"name":"http-internal","protocol":"TCP"}]
```

We can test port 8080 by using `curl` within the pod itself, targetting localhost:

```console
$ k exec --stdin --tty sas-studio-app-75b59846b4-2s69z -- curl -k https://localhost:8080/SASStudio/
{"timestamp":1659346441951,"status":401,"error":"Unauthorized","message":"","path":"/SASStudio/"}
```

... and, unsurprisingly, it works (we will not use authorization in this examples to keep things simple: getting the reply is enough to test the network flow).

To test if we can reach the service from outside the pods, we will now test from
another one. Using a container image with network tools
([jedrecord/nettools](https://github.com/jedrecord/nettools)), we spin up a pod
and login interactively, then try to access SAS Studio using the IP and port.

```console
$ kubectl run -it --rm  --image=jrecord/nettools nettools --restart=Never
If you don't see a command prompt, try pressing enter.
[root@nettools /]# curl -k https://10.244.2.21:8080/SASStudio/
{"timestamp":1659349707524,"status":401,"error":"Unauthorized","message":"","path":"/SASStudio/"}[root@nettools /]#

```

It also works, so it seems that we just need that pod IP address and port and we
are good to go... but this isn't true!

The reason is that the pod IP is dynamically assigned by Kubernetes, from a
range of addresses reserved for Pods in the node. The most direct implication of
this is that the IP information should be considered ephemeral, because it
actually is: whenever the pod is restarted it will be assigned a different IP.

One typical reason for this would be a restart of the cluster, but since we are
not concerned with end-user availability, we can be more blunt and delete the
SAS Studio pod we've been using until now

```console
$ k delete pod sas-studio-app-75b59846b4-2s69z
pod "sas-studio-app-75b59846b4-2s69z" deleted
```

Kubernetes will automatically launch a new one, this being one of the reason why
Kubernetes is a container orchestration platform:

```console
$ k get pod|grep sas-studio-app
sas-studio-app-75b59846b4-gnghj                                   0/1     Running      0             32s
```

It looks identical, but a closer look will show a different IP address:

```console
$ k get pod sas-studio-app-75b59846b4-gnghj -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP             NODE                                NOMINATED NODE   READINESS GATES
sas-studio-app-75b59846b4-gnghj   1/1     Running   0          2m11s   10.244.5.121 aks-stateless-51044827-vmss000004   <none>           <none>
```

When we redo our connection test, the command that previously worked now... doesn't:

```console
root@nettools /]# curl -k https://10.244.2.21:8080/SASStudio/
curl: (7) Failed to connect to 10.244.2.21 port 8080: No route to host
```

Using the new Pod IP will work:

```console
[root@nettools /]# curl -k https://10.244.5.121:8080/SASStudio/
{"timestamp":1659353103484,"status":401,"error":"Unauthorized","message":"","path":"/SASStudio/"}[root@nettools /]#
```

We have so far determined a couple of important points:

- SAS Viya has many Pods running, one of which is the one for SAS Studio.
- The SAS Studio Pod is based on a container image that runs the Java
  application (Spring-based).
- The pod gets assigned a random IP from the Pod CIDR range when it starts,
  every time it starts.
- We can reach that IP from other pods in the cluster, but we can't depend on
  the IP because it changes.

It should be noted that it's not only the IP that is nonpermanent: Pods
themselves are considered nonpermanent resoures, and they can (and are) created
and destroyed dynamically by a Deployment resource.

> **Note**
> We represented some other resources in the initial diagram: a Secret
> (that stores certificate tokens for use in the pod), a ConfigMap (that makes
> certificates available to use in the pod), and a Deployment (the defines all
> of these in a single logical unit, and through a ReplicaSet makes sure that
> the correct amount of pods are running at all times); they will not appear in
> the next diagrams to simplify things and keep the focus on the network flow.

We need a way to keep track of the IPs so that pods can interact safely.
Fortunately, Kubernetes has a way to address this, and much more: **Services**.

## Internal load-balancing : the Service

![The SAS Studio Service](./svc-view.png)

A [Kubernetes
Service](https://kubernetes.io/docs/concepts/services-networking/service/) is a
Kubernetes resource that allows us to define a set of target pods, usually
determined by a _selector_, and that has a stable, cluster-wide IP.

Continuing with our SAS Studio journey, SAS Viya includes a Service for it (and,
in general, a Service for all the other components), the details of which
include the IP address:

```console
$ k get service sas-studio-app -o wide
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
sas-studio-app   ClusterIP   10.0.72.50   <none>        443/TCP   3d    app.kubernetes.io/name=sas-studio-app,sas.com/deployment=sas-viya

```

This Service has the Cluster IP of 10.0.72.50 - notice that it's within the
Service CIDR of the cluster, different than the one for Pods - and listens in
port 443. The selector targets pods that have two labels: one indicating that
they are part of SAS Viya, and the other that they are named `sas-studio-app`.

A closer look into the Service definition will make the connection between it
and the pods we have seen in the previous section clearer:

```console
$ k describe service sas-studio-app
Name:              sas-studio-app
[...]
Selector:          app.kubernetes.io/name=sas-studio-app,sas.com/deployment=sas-viya
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.0.72.50
IPs:               10.0.72.50
Port:              http  443/TCP
TargetPort:        http/TCP
Endpoints:         10.244.5.121:8080
Session Affinity:  None
Events:            <none>
```

This Service listens on 10.0.72.50:443, and directs traffic to pods that match
the selector at port 8080. Looking closely we can find the IP of the SAS Studio
pod, from last section, in the `Endpoints` entry.

Returning to our `nettools` pod, we can test this flow:

```console
[root@nettools /]# curl -k https://10.0.72.50/SASStudio/
{"timestamp":1659353286417,"status":401,"error":"Unauthorized","message":"","path"
```

The Service IP of the Service, which can be reached anywhere in the cluster,
sends the traffic to the SAS Studio pod as expected; note that we are now using
the default HTTPS port since that's the one where the Service listens to.

One additional feature of Services are the built-in DNS support: while the IP is
stable throughout the Service lifecycle, a DNS `A` entry is created
automatically using the form `<service-name>.<namespace>.<cluster-name>`:

```console
[root@nettools /]# curl -k https://sas-studio-app.sas-viya.svc.cluster.local/SASStudio/
{"timestamp":1659353439229,"status":401,"error":"Unauthorized","message":"","path":"/SASStudio/"}
```

We now have a stable way to make our requests reach the SAS Studio pod. In the
case of applications with multiple replicas, the Service will keep track of the
health of the endpoints that match the selector and load-balance traffic
automatically. Whenever pods get destroyed and rescheduled (for example, a
cluster node goes down and pods are automatically restarted in another node),
the Service will update its configuration to point to the right place.

In this stretch of our path we have discovered some interesting tidbits:

- Services redirect and load-balance traffic using a selector.
- Services keep track of the health of pods, updating the backend IPs dynamically.
- Services provide us with a stable CLuster IP and a cluster internal DNS
  hostname for the service.

With this, we are almost at the end of our journey, but there's something
important missing: Services provide a Cluster IP which is internal, so they are
not directly accessable by the outside world. For that, we will need something
else: an Ingress.

## Exposing to the outside world: the Ingress

![The SAS Studio Ingress](./ing-view.png)

We went from Pods, to Services, and now we reach another layer: [the
Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/). In
it's simpler form, an Ingress manages external access, routing requests coming
from the outside to Services inside the cluster.

Just as for Services, SAS Viya deploys a number of Ingresses, including one for SAS Studio:

```console
$ k get ingress sas-studio-app
NAME             CLASS   HOSTS                                               ADDRESS         PORTS     AGE
sas-studio-app   nginx   doriath.beleriand.cloud,*.doriath.beleriand.cloud   20.103.174.39   80, 443   3d1h
```

This information makes it clear we are dealing with exposing our resources: the
`HOSTS` field indicates the externally-accessible hostname that we must define
when deploying SAS Viya, and it has an `ADDRESS` field with a public IP address
(in this case, sincce it could also be an IP in a private range, but it would
indicate the IP associated with the external hostname).

Describing this specific Ingress enables us to get a better understanding on
what goes on:

```console
$ k describe ingress sas-studio-app
Name:             sas-studio-app
Namespace:        sas-viya
Address:          20.103.174.39
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  sas-ingress-certificate-24mffmch6d terminates doriath.beleriand.cloud,*.doriath.beleriand.cloud
Rules:
  Host                       Path  Backends
  ----                       ----  --------
  doriath.beleriand.cloud
                             /SASStudio   sas-studio-app:443 (10.244.5.121:8080)
  *.doriath.beleriand.cloud
                             /SASStudio   sas-studio-app:443 (10.244.5.121:8080)
[...]
```

The above configuration can be read like this: whenever there is a request to
`https://doriath.beleriand.cloud/SASStudio`, as indicated by the `Host` and
`Path`, traffic should be sent to the `sas-studio-app` Service, listening on
port 443. Note the pod IP address and port between parenthesis: this is
additional information that is being displayed, and not a part of the Ingress
definition. It's there to facilitate our view on the active path that requests
will take by adding the information of the active Service endpoints.

The Ingress is, ultimately, a rule that determines how traffic will be dealt
with, but a rule needs something to implement it: in Kubernetes, this is the
Ingress Controller. In SAS Viya one of the requirements is an [NGINX Ingress
controller](https://github.com/kubernetes/ingress-nginx).


## The NGINX Ingress controller

![The NGINX Ingress Controller](./ingc-view.png)

If the idea of defining rules to direct HTTP traffic seems something that an
HTTP server or proxy would do, well, that's because that's true: the [Ingress
controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
is a Layer 7 proxy that directs requests to the backend, and specifically the
NGINX Ingress controller uses NGINX as a reverse proxy and load-balancer.

As for NGINX itself, it runs in a pod (or more than one, two are a common
setting) in its own namespace:

```console
$ k get pods -n ingress-nginx -o wide
NAME                                        READY   STATUS    RESTARTS   AGE   IP           NODE                             NOMINATED NODE   READINESS GATES
ingress-nginx-controller-84cc585d9c-jgp7g   1/1     Running   0          18h   10.244.3.6   aks-system-16375213-vmss000004   <none>           <none>

```

The reason we are talking about "Ingress" and "Ingress controller" is mostly
because, when NGINX is used in this role in Kubernetes, the way we can configure
it is different and completely integrated with the Kubernetes: instead of
manually adding rules to the NGINX configuration files, we specify them as
Ingress rules and they get automatically added to the configuration, as we can
easily see by checking the `nginx.conf` file in the pod:

```console
$ k  exec --stdin --tty pod/ingress-nginx-controller-84cc585d9c-jgp7g -n ingress-nginx -- bash
bash-5.1$ grep SASStudio /etc/nginx/nginx.conf
		location ~* "^/SASStudio/" {
			set $location_path  "/SASStudio";
		location ~* "^/SASStudio" {
			set $location_path  "/SASStudio";
		location ~* "^/SASStudio/" {
			set $location_path  "/SASStudio";
		location ~* "^/SASStudio" {
			set $location_path  "/SASStudio";
```

Additionally, the NGINX Ingress controller automates other aspects which are
needed when working in a Kubernetes context, and there is one aspect which is
important to highlight: the way it interact with cloud environments to provide a
public endpoint IP.

## Reaching the Ingress: cloud Load Balancers


The last step in our journey, this is the first an only which is outside the
Kubernetes cluster itself.

We've seen that to expose the internal Services, we use an NGINX Ingress
controller that will be configured, through Ingress resources, to direct traffic
to different Services.

NGINX pods can be deployed in different nodes in the cluster (just like any
other pod), so any external connectivity needs to address how to provide a
stable external IP address that load-balances all the cluster nodes: we need a
single, external endpoint that will send the traffic to the NGINX pod in the cluster.

For [cloud deployment, and specifically for
AKS](https://kubernetes.github.io/ingress-nginx/deploy/#cloud-deployments),
NGINX automatically creates an external Load Balancer by using a `LoadBalancer`
Service:

```console
$ k get svc ingress-nginx-controller -n ingress-nginx -o wide
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE    SELECTOR
ingress-nginx-controller   LoadBalancer   10.0.200.113   20.103.174.39   80:30623/TCP,443:30954/TCP   3d3h   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
```

This type of Service should not be mistaken by the `ClusterIP` types we have
already covered: a Load Balancer Service will interact with the Cloud
infrastructure and automate the creation of an external load balancer, and will
store the IP that is returned. As you can see, the `EXTERNAL-IP` of this Service
is the same that appears in all Ingress rules, and is indicated in the
`LoadBalancer Ingress` field as well:

```console
$ k describe svc ingress-nginx-controller -n ingress-nginx
Name:                        ingress-nginx-controller
Namespace:                   ingress-nginx
[...]
Selector:                    app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
Type:                        LoadBalancer
IP Family Policy:            SingleStack
IP Families:                 IPv4
IP:                          10.0.200.113
IPs:                         10.0.200.113
LoadBalancer Ingress:        20.103.174.39
Port:                        http  80/TCP
TargetPort:                  http/TCP
NodePort:                    http  30623/TCP
Endpoints:                   10.244.3.6:80
Port:                        https  443/TCP
TargetPort:                  https/TCP
NodePort:                    https  30954/TCP
Endpoints:                   10.244.3.6:443
Session Affinity:            None
External Traffic Policy:     Local
HealthCheck NodePort:        31081
[...]
```

Just like any Service, it has a selector, which in this case directs traffic to
the NGINX Ingress controller pods (the `Endpoints` field has the IP of the NGINX
pod from last section). The internal IP, 10.0.200.113, can also be used to send
traffic to NGINX, but obviously will only work inside the cluster.

In terms of Azure, the topology will be something like this:

![Azure Load Balancer](./azure-lb.png)

The cloud load balancer will check the status of all nodes by connecting to
http://<node-external-ip>:31081/healthz (this can be seen in the _Health Probes_
blade in Azure). The nodes where the ingress pod is running will answer with
`HTTP 200`, and traffic will be sent to that node's external IP on the ports
where it's listening - 30623(HTTP) and 30954 (HTTPS).

> **Note** there is a subtle detail that we are not showing in our diagrams or
> descriptions: when looking into the Azure Load Balancer configuration in AKS,
> the target port for the backend is set at 80 (HTTP) and 443 (HTTPS)... but the
> LoadBalancer Service definition creates NodePorts, in each node, listening on
> 30623(HTTP) and 30954 (HTTPS). This is a _very_ lightly documented feature of
> AKS: what happens is that in the Kubernetes nodes, at the host Linux level,
> there are `iptables` rules that do a DNAT on those ports, when they have the
> load balancer IP as the target: in short, they transparently redirect traffic
> coming from the load balancer to the Service ports. This is very hard to test
> and can only be detected by looking into the `iptables` chains at the host
> level (check https://github.com/kubernetes/kubernetes/issues/58759 for more
> details).


This load balancer IP is the final destination of our analysis: this is what we
will use in our browser to access SAS Viya, in general, and SAS Studio in
particular. We are now ready to make our journey back to the pod, now with a
clearer understanding of what it entails.


# SAS Viya networking, Or, There and Back Again

Having gone through this step-by-step exploration of the different components,
we should now be more prepared for a diagram that goes a bit more in-depth, and
that we can use as a reference:

![SAS Viya on AKS](./sas-viya-k8s.png)


Starting from the outside - for example, an end-user browser trying to access
SAS Studio - this is how the requests get into the SAS Studio pod:


1. Someone makes a request to https://doriath.beleriand.cloud/SASStudio . This
   hostname was determined from the start as the one for SAS Viya.
2. The hostname is converted into an IP address, almost always through DNS (but
   could be through `/etc/hosts` as well).
3. This IP address, `20.103.174.39`, is the IP of the frontend of Azure Load
   Balancer, which has rules to direct traffic on ports 80 and 443 towards all
   nodes of the Kubernetes cluster. This Load Balancer was created because of
   the `LoadBalancer` Service that was added by the NGINX Ingress controller,
   which interacted with Azure to spin up the cloud load balancer, and stored
   the IP that was configured as the frontend.
4. The load balancer determines which nodes in the cluster are able to receive
   the request, using a health probe, and send the request to the port defined
   in the LoadBalancer Service on that node's external IP.
5. The request arrives at the cluster, and is sent to an NGINX pod.
6. The path `/SASStudio` is processed as per the NGINX rules, which are
   determined by Ingress resources: in this case, there is an Ingress
   definition that says that traffic with that path component should be sent to
   the `sas-studio-app` Service, on port 443.
7. The `sas-studio-app` Service receives the request and sends it to the
   configured endpoints, which are the pods that fit a specific selector. In
   this case, the `sas-studio-app` pod, in whatever internal IP it currently has,
   on port 8080.
8. The `sas-studio-app` pod (named something like
   `sas-studio-app-75b59846b4-gnghj`) receives the request on its IP, port 8080,
   where the SAS Studio application is running.

After receiving the request, the SAS Studio application sends the reply: note
that this is not up the same circuit, and will, in the most simple cases, be
sent through the external IP of the node where the pod is running, which is
possibly then directed to an outgoing IP in the same cloud load balancer (this
is the situation in the AKS cluster used here).

# Final words

TBD

# Possible additions

- A reference to SSL termination in the Ingress.
- A reference to Service Meshes
