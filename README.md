### Monitoring Namespace ###
We are going to install the monitoring components into a "monitoring"
namespace.  While this is not necessary, it does show "best practices"
in organizing applications by namespace rather than deploying
everything into the default namespace.


First, create the monitoring namespace: `kubectl create -f
monitoring-namespace.yaml`.

You can now list the namespaces by running `kubectl get namespaces`
and you should see something similar to:

```
NAME          STATUS    AGE
default       Active    1h
kube-system   Active    1h
monitoring    Active    1h
```

## Deploying Prometheus and Grafana ##

Let's step through deploying Prometheus.  



### Prometheus Configuration ###
Prometheus will get its configuration from a
[Kubernetes ConfigMap](http://kubernetes.io/docs/user-guide/configmap/).
This allows us to update the configuration separate from the image.


Look at prometheus-config.yaml. The
relevant part is in `data/prometheus.yml`.  This is just a [Prometheus
configuration](https://prometheus.io/docs/operating/configuration/)
inlined into the Kubernetes manifest. Note that we are using the in-cluster
[Kubernetes service account](http://kubernetes.io/docs/user-guide/service-accounts/) to access the Kubernetes API.

To deploy this to the cluster run `kubectl create -f prometheus-config.yaml`.  
You can view this by running 

`kubectl get configmap --namespace=monitoring prometheus-config -o yaml`. 

You can also see this in the Kubernetes Dashboard.


### Prometheus Pod ###
We will use a single Prometheus
[pod](http://kubernetes.io/docs/user-guide/pods/) for this demo. 
Take a look at [prometheus-deployment.yaml](./prometheus-deployment.yaml).
This is a [Kubernetes Deployment](http://kubernetes.io/docs/user-guide/deployments/) that describes the image to use for
the pod, resources, etc.  Note:

* In the metadata section, we give the pod a label with a key of
`name` and a value of `prometheus`. This will come in handy later.
* In annotations, we set a couple of key/value pairs that will
actually allow Prometheus to autodiscover and scrape itself.
* We are using an
  [emptyDir volume](http://kubernetes.io/docs/user-guide/volumes/#emptydir)
  for the Prometheus data.  This is basically a temporary directory
  that will get erased on every restart of the container.  For a demo
  this is fine, but we'd do something more persistent for other use
  cases.

Deploy the deployment by running `kubectl create -f prometheus-deployment.yaml`.  
You can see this by running `kubectl get deployments --namespace=monitoring`.

### Prometheus Service ###

Now that we have Prometheus deployed, we actually want to get to the
UI.  To do this, we will expose it using a
[Kubernetes Service](http://kubernetes.io/docs/user-guide/services/).

In prometheus-service.yaml, there are a
few things to note:

* The label selector searches for pods that have been labeled with
`name: prometheus` as we labeled our pod in the deployment.
* We are exposing port 9090 of the running pods.
* We are using a "NodePort."  This means that Kubernetes will open a
port on each node in our cluster. You can query the API to get this
port.

Create the service by running `kubectl create -f prometheus-service.yaml`.  
You can then view it by running `kubectl get services --namespace=monitoring prometheus -o yaml`.



One thing to note is that you will see something like `nodePort:
30827` in the output.  We could access the service on that port on any
node in the cluster.  

From the Prometheus console, you can explore the metrics is it
collecting and do some basic graphing.  You can also view the
configuration and the targets. Click Status->Targets and you should
see the Kubernetes cluster and nodes.  You should also see that
Prometheus discovered itself under `kubernetes-pods`

### Deploying Grafana ###

You can deploy grafana by creating its deployment and service by
running `kubectl create -f grafana-deployment.yaml` 
and `kubectl create -f grafana-service.yaml`.

You can then view it by running `kubectl get services --namespace=monitoring grafana -o yaml`


Feel free to explore via the kubectl command line and/or the Dashboard.

Username is `admin` and password is also `admin`.

Let's add Prometheus as a datasource.
* Click on the icon in the upper
left of grafana and go to "Data Sources".
* Click "Add data
source".
* For name, just use "prometheus"
* Select "Prometheus" as the type
* For the URL, we will actual use [Kubernetes DNS service
  discovery](http://kubernetes.io/docs/user-guide/services/#dns). So,
  just enter `http://prometheus:9090`. This means that grafana will
  lookup the `prometheus` service running in the same namespace as it
  on port 9090.

Create a New dashboard by clicking on the upper-left icon and
selecting Dashboard->New.  Click the green control and add a graph
panel.  Under metrics, select "prometheus" as the datasource. For the
query, use `sum(container_memory_usage_bytes) by (kubernetes_pod_name)`.  Click
save. This graphs the memory used per pod.

### Weave Scope ###

Weave Scope is an open source tool that helps you monitor and visualize your cluster. It is currently very beta, but I think it has a lot of potential!
Running it is also super easy.

### Step 1: Deploy ###

$ kubectl apply -f weavescope.yaml

### Step 2: Connect ###

$ kubectl port-forward $(kubectl get pod --selector=weave-scope-component=app -o jsonpath={.items..metadata.name}) 4040

### Step 3: Open ###

http://localhost:4040/




