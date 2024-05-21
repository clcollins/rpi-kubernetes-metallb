#This repository is archived and will no longer receive updates.

# Install a LoadBalancer on your Raspberry Pi homelab with MetalLB

Kubernetes is designed to integrate with the major cloud providers' load-balancer offerings to provide public IP addresses and direct traffic into the cluster. Some professional network equipment manufacturers offer controllers to integrate their physical load-balancing products into Kubernetes installations in private datacenters, as well. For an enthusiast running a Kubernetes cluster at home, though, neither of these solutions is very helpful.

Kubernetes does not have a network load-balancer implementation built in. Bare metal clusters, such as a [Kubernetes cluster on Raspberry Pis installed for a private-cloud homelab](https://opensource.com/article/20/6/kubernetes-raspberry-pi) - or really an cluster deployed outside of a public cloud and lacking expensive professional hardware, need another solution. MetalLB fulfills this niche, both for enthusiasts and large-scale deployments.

MetalLB is a network load-balancer, and can expose cluster services on a dedicated IP addresses on the network, allowing external clients to connect to services inside the Kubernetes cluster. It does this either via [layer 2 (data-link)](https://en.wikipedia.org/wiki/Data_link_layer) using ARP ([Address Resolution Protocol](https://en.wikipedia.org/wiki/Address_Resolution_Protocol)), or [layer 4 (transport)](https://en.wikipedia.org/wiki/Transport_layer), using BGP ([Border Gateway Protocol](https://en.wikipedia.org/wiki/Border_Gateway_Protocol)).

While Kubernetes does have something called [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), which allows HTTP and HTTPS traffic to be exposed outside of the cluster, this _only_ supports HTTP or HTTPS traffic, while MetalLB can support any network traffic. It is more of an apples-to-oranges comparison, however, because MetalLB is provides resolution of an unassigned IP address to a particular cluster node, and assigns that IP to a Service, while Ingress uses a specific IP address, and internally routes HTTP or HTTPS traffic to a Service or Services based on routing rules.

**NOTE TO SETH - Can I get technical review of that last bit? Is it fair to say layer 2/layer 4 supports "any network traffic", or is there a better way to put that?  As opposed to HTTP/HTTPS at the application layer?**

MetalLB can be setup in just a few steps works especially well in private homelab clusters, and within Kubernetes clusters, behaves the same as public cloud load balancer integrations.  This is great for both education purposes (learning how the technology works) and makes it easier to "lift-and-shift" workloads between on-premise and cloud environments.

## ARP vs BGP

As mentioned above, MetalLB works either via ARP or BGP to resolve IP addresses to specific hosts. In simplified terms, this means when a client attempts to connect to a specific IP, it will ask "which host has this IP?" and will receive a response pointing it to the correct host (the MAC address of the host).

With ARP, the request is broadcast to the entire network, and a host that knows which MAC address  has that IP address responds to the request - in this case MetalLB' responds with the answer directing the client to the correct node.

With BGP, each "peer" maintains a table of routing information directing clients to the host handling a particular IP for IPs and hosts the peer knows about, and advertises this information to it's peers. When configured for BGP, MetalLB peers each of the nodes in the cluster with the network's router, allowing the router to direct clients to the correct host.

In both instances, once the traffic has arrived at a host, Kubernetes takes over directing the traffic to the correct pods.

For our exercise today, we'll be using ARP. Consumer grade routers don't (at least easily) support BGP, and even higher-end consumer or professional routers that do support BGP can be difficult to setup. ARP, especially in a small home network, can be just as useful and requires no configuration on the network to work. It is considerably easier to implement.

## Install MetalLB

Installing MetalLB is straightforward. Two manifests are downloaded or copied from the MetalLB Github repository, and applied to Kubernetes. These two manifests create the namespace MetalLB's components will be deployed to, and the components themselves: the MetalLB controller, a "speaker" daemonset and service accounts.

### Install the components

Once the components are created a random secret is generated to allow encrypted communication between the speakers.

_Note: These steps are available on the MetalLB website as well._

The manifests with the MetalLB components needed are the following:

* [https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml](https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml)
* [https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml](https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml)

They can be downloaded and applied to the Kubernetes cluster using the `kubectl apply` command, either locally or directly from the web:

```shell
# Verify the contents of the files, then download and pipe then to kubectl with curl
# (output omitted)
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
```

With the manifests applied, create a random Kubernetes secret for the speakers to use for their encrypted communications:

```shell
# Create a secret for encrypted speaker communications
$ kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Completing the steps above will create and start all the MetalLB components, but they will not do anything until they are configured. To configure MetalLB, we need to create a configMap that describes the pool of IP Addresses to be used by the load balancer.

### Configure the address pools

MetalLB needs one last bit of setup: a configMap with details of the addresses it can assign to the Kubernetes Service LoadBalancers. However, there is a small consideration. The addresses in use do not need to be bound to specific hosts in the network, but they must be free for MetalLB to use, and not assigned to other hosts.

In my home network, IP addresses are assigned by the DHCP server being run by my router. This DHCP server should not attempt to assign the addresses that MetalLB will be using. Most consumer routers allow you to decide how large your subnet will be, and can be configured to assign only a subset of IPs in that subnet to hosts via DHCP.

In my network, I am using the subnet 192.168.2.1/24, and I decided to give half of the IPs to MetalLB. The first half of the subnet consist of IP addresses from 192.168.2.1 - 192.168.2.126.  This range can be represented by a /25 subnet: 192.168.2.1/25.  The second half of the subnet can similarly be represented by a /25 subnet: 192.168.2.128/25. Each half contains 126 IPs - more than enough for both hosts and Kubernetes services. Make sure to decide on subnets appropriate to your own network, and configure your router and MetalLB appropriately.

After configuring the router to ignore addresses in the 192.168.2.128/25 subnet (or whatever subnet you are using), create a configMap to tell MetalLB to use that pool of addresses:

```shell
# Create the config map
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: address-pool-1
      protocol: layer2
      addresses:
      - 192.168.2.128/25
EOF
```

The example configMap above uses CIDR notation, but the list of addresses can also be specified as a range:

```YAML
addresses:
  - 192.168.2.128-192.168.2.254
```

Once the configMap has been created, MetalLB will be active. Time to try it out!

## Testing MetalLB

The new MetalLB configuration can be tested by creating an example webservice. We can use one we've already tried out for a previous article: [Kube Verify from "Build a Kubernetes cluster with the Raspberry Pi"](https://opensource.com/article/20/6/kubernetes-raspberry-pi). If you created your Kubernetes cluster on Raspberry Pis following that article, you may already have a Kube Verify service running, and can skip down to the section creating a "LoadBalancer" type service.

We are going to use the same image to test that MetalLB is working as expected: `quay.io/clcollins/kube-verify:01`. This image contains an Nginx server listening for requests on port 8080. You can view the Containerfile used to create the image [here](https://github.com/clcollins/homelabCloudInit/blob/master/simpleCloudInitService/data/Containerfile). If you want, you can build your own container image from the Containerfile and use that to test, instead.

If you do not already have a `kube-verify` namespace, create one with the `kubectl` command:

```shell
# Create a new namespace
$ kubectl create namespace kube-verify
# List the namespaces
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   63m
kube-node-lease   Active   63m
kube-public       Active   63m
kube-system       Active   63m
metallb-system    Active   21m
kube-verify       Active   19s
```

With the namespace created, now create a deployment in that namespace:

```shell
# Create a new deployment
$ cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-verify
  namespace: kube-verify
  labels:
    app: kube-verify
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kube-verify
  template:
    metadata:
      labels:
        app: kube-verify
    spec:
      containers:
      - name: nginx
        image: quay.io/clcollins/kube-verify:01
        ports:
        - containerPort: 8080
EOF
deployment.apps/kube-verify created
```

Now, expose the deployment by creating a LoadBalancer-type Kubernetes service. If you already have a service named "kube-verify", this will replace the existing one:

```shell
# Create a LoadBalancer service for the kube-verify deployment
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: kube-verify
  namespace: kube-verify
spec:
  selector:
    app: kube-verify
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
EOF
```

You could accomplish the same thing with the `kubectl expose` command: `kubectl expose deployment kube-verify -n kube-verify --type=LoadBalancer --target-port=8080 --port=80`. MetalLB is listening for services of type "LoadBalancer", and immediately assigns an "external ip" (an IP chosen from the range you selected when setting up MetalLB). View the new service, and the external IP address assigned to it by MetalLB, with the `kubectl get service` command:

```shell
# View the new kube-verify service
$ kubectl get service kube-verify -n kube-verify
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kube-verify   LoadBalancer   10.105.28.147   192.168.2.129   80:31491/TCP   4m14s

# Look at the details of the kube-verify service
$ kubectl describe service kube-verify -n kube-verify
Name:                     kube-verify
Namespace:                kube-verify
Labels:                   app=kube-verify
Annotations:              <none>
Selector:                 app=kube-verify
Type:                     LoadBalancer
IP:                       10.105.28.147
LoadBalancer Ingress:     192.168.2.129
Port:                     <unset>  80/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31491/TCP
Endpoints:                10.244.1.50:8080,10.244.1.51:8080,10.244.2.36:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason        Age    From                Message
  ----    ------        ----   ----                -------
  Normal  IPAllocated   5m55s  metallb-controller  Assigned IP "192.168.2.129"
  Normal  nodeAssigned  5m55s  metallb-speaker     announcing from node "gooseberry"
```

You will notice in the output from the `kubectl describe` command the events at the bottom, where MetalLB has assigned an ip address (yours will vary) and is "announcing" the assignment from one of the nodes in your cluster (again, yours will vary). It also describes the port, the external port you can access the service from (80), the target port inside the container (port 8080), and a node port through which the traffic will route (31491).  The end result is that the Nginx server running in the pods of the "kube-verify" service is accessible from the load-balanced IP, on port 80, from anywhere on your home network.

For example, on my network, the service was exposed on `http://192.168.2.129:80`, and I can curl that IP from my laptop, on the same network:

```shell
# Verify that you receive a response from Nginx on the load-balanced IP
$ curl 192.168.2.129
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN"
    "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" lang="en">
<head>
  <title>Test Page for the HTTP Server on Fedora</title>
(further output omitted)
```

## MetalLB FTW

MetalLB is a great choice of load-balancer for a home Kubernetes cluster. It allows you to assign real IPs from your home network to services running in your cluster, and access them from other hosts on your home network. These services can even be exposed outside the network by port forwarding traffic through your home router (but, please be careful with this!). MetalLB easily replicates cloud-provider like behavior at home, on bare-metal computers, Raspberry Pi-based clusters and even virtual machines, making it easy to "lift-and-shift" workloads to the cloud, or just familiarize yyourself with how they work. Best of all, MetalLB is easy and convenient, and makes accessing the services running in your cluster a breeze.

Have you used MetalLB, or do you use another load balancer solution? Are you using primarily Nginx or HAProxy ingress? Let me know in the comments!
