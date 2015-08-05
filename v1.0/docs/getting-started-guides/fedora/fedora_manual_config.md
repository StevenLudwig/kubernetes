---
layout: docwithnav
---
<!-- BEGIN MUNGE: UNVERSIONED_WARNING -->


<!-- END MUNGE: UNVERSIONED_WARNING -->
Getting started on [Fedora](http://fedoraproject.org)
-----------------------------------------------------

**Table of Contents**

- [Prerequisites](#prerequisites)
- [Instructions](#instructions)

## Prerequisites

1. You need 2 or more machines with Fedora installed.

## Instructions

This is a getting started guide for Fedora.  It is a manual configuration so you understand all the underlying packages / services / ports, etc...

This guide will only get ONE node (previously minion) working.  Multiple nodes require a functional [networking configuration](../../admin/networking.html) done outside of Kubernetes.  Although the additional Kubernetes configuration requirements should be obvious.

The Kubernetes package provides a few services: kube-apiserver, kube-scheduler, kube-controller-manager, kubelet, kube-proxy.  These services are managed by systemd and the configuration resides in a central location: /etc/kubernetes.  We will break the services up between the hosts.  The first host, fed-master, will be the Kubernetes master.  This host will run the kube-apiserver, kube-controller-manager, and kube-scheduler.  In addition, the master will also run _etcd_ (not needed if _etcd_ runs on a different host but this guide assumes that _etcd_ and Kubernetes master run on the same host).  The remaining host, fed-node will be the node and run kubelet, proxy and docker.

**System Information:**

Hosts:

```
{% raw %}
fed-master = 192.168.121.9
fed-node = 192.168.121.65
{% endraw %}
```

**Prepare the hosts:**
    
* Install Kubernetes on all hosts - fed-{master,node}.  This will also pull in docker. Also install etcd on fed-master.  This guide has been tested with kubernetes-0.18 and beyond.
* The [--enablerepo=update-testing](https://fedoraproject.org/wiki/QA:Updates_Testing) directive in the yum command below will ensure that the most recent Kubernetes version that is scheduled for pre-release will be installed. This should be a more recent version than the Fedora "stable" release for Kubernetes that you would get without adding the directive. 
* If you want the very latest Kubernetes release [you can download and yum install the RPM directly from Fedora Koji](http://koji.fedoraproject.org/koji/packageinfo?packageID=19202) instead of using the yum install command below.

{% highlight sh %}
{% raw %}
yum -y install --enablerepo=updates-testing kubernetes
{% endraw %}
{% endhighlight %}

* Install etcd and iptables

{% highlight sh %}
{% raw %}
yum -y install etcd iptables
{% endraw %}
{% endhighlight %}

* Add master and node to /etc/hosts on all machines (not needed if hostnames already in DNS). Make sure that communication works between fed-master and fed-node by using a utility such as ping.

{% highlight sh %}
{% raw %}
echo "192.168.121.9	fed-master
192.168.121.65	fed-node" >> /etc/hosts
{% endraw %}
{% endhighlight %}

* Edit /etc/kubernetes/config which will be the same on all hosts (master and node) to contain:

{% highlight sh %}
{% raw %}
# Comma separated list of nodes in the etcd cluster
KUBE_MASTER="--master=http://fed-master:8080"

# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow_privileged=false"
{% endraw %}
{% endhighlight %}

* Disable the firewall on both the master and node, as docker does not play well with other firewall rule managers.  Please note that iptables-services does not exist on default fedora server install.

{% highlight sh %}
{% raw %}
systemctl disable iptables-services firewalld
systemctl stop iptables-services firewalld
{% endraw %}
{% endhighlight %}

**Configure the Kubernetes services on the master.**

* Edit /etc/kubernetes/apiserver to appear as such.  The service_cluster_ip_range IP addresses must be an unused block of addresses, not used anywhere else.  They do not need to be routed or assigned to anything.

{% highlight sh %}
{% raw %}
# The address on the local server to listen to.
KUBE_API_ADDRESS="--address=0.0.0.0"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd_servers=http://127.0.0.1:4001"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# Add your own!
KUBE_API_ARGS=""
{% endraw %}
{% endhighlight %}

* Edit /etc/etcd/etcd.conf,let the etcd to listen all the ip instead of 127.0.0.1, if not, you will get the error like "connection refused"

{% highlight sh %}
{% raw %}
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:4001"
{% endraw %}
{% endhighlight %}

* Start the appropriate services on master:

{% highlight sh %}
{% raw %}
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do
	systemctl restart $SERVICES
	systemctl enable $SERVICES
	systemctl status $SERVICES
done
{% endraw %}
{% endhighlight %}

* Addition of nodes:

* Create following node.json file on Kubernetes master node:

{% highlight json %}
{% raw %}
{
    "apiVersion": "v1",
    "kind": "Node",
    "metadata": {
        "name": "fed-node",
        "labels":{ "name": "fed-node-label"}
    },
    "spec": {
        "externalID": "fed-node"
    }
}
{% endraw %}
{% endhighlight %}

Now create a node object internally in your Kubernetes cluster by running:

{% highlight console %}
{% raw %}
$ kubectl create -f ./node.json

$ kubectl get nodes
NAME                LABELS              STATUS
fed-node           name=fed-node-label     Unknown
{% endraw %}
{% endhighlight %}

Please note that in the above, it only creates a representation for the node
_fed-node_ internally. It does not provision the actual _fed-node_. Also, it
is assumed that _fed-node_ (as specified in `name`) can be resolved and is
reachable from Kubernetes master node. This guide will discuss how to provision
a Kubernetes node (fed-node) below.

**Configure the Kubernetes services on the node.**

***We need to configure the kubelet on the node.***

* Edit /etc/kubernetes/kubelet to appear as such:

{% highlight sh %}
{% raw %}
###
# Kubernetes kubelet (node) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname_override=fed-node"

# location of the api-server
KUBELET_API_SERVER="--api_servers=http://fed-master:8080"

# Add your own!
#KUBELET_ARGS=""
{% endraw %}
{% endhighlight %}

* Start the appropriate services on the node (fed-node).

{% highlight sh %}
{% raw %}
for SERVICES in kube-proxy kubelet docker; do 
    systemctl restart $SERVICES
    systemctl enable $SERVICES
    systemctl status $SERVICES 
done
{% endraw %}
{% endhighlight %}

* Check to make sure now the cluster can see the fed-node on fed-master, and its status changes to _Ready_.

{% highlight console %}
{% raw %}
kubectl get nodes
NAME                LABELS              STATUS
fed-node          name=fed-node-label     Ready
{% endraw %}
{% endhighlight %}

* Deletion of nodes:

To delete _fed-node_ from your Kubernetes cluster, one should run the following on fed-master (Please do not do it, it is just for information):

{% highlight sh %}
{% raw %}
kubectl delete -f ./node.json
{% endraw %}
{% endhighlight %}

*You should be finished!*

**The cluster should be running! Launch a test pod.**

You should have a functional cluster, check out [101](../../../docs/user-guide/walkthrough/README.html)!


<!-- TAG IS_VERSIONED -->


<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/docs/getting-started-guides/fedora/fedora_manual_config.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->
