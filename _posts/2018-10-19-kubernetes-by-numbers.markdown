---
layout: post
title:  "Kubernetes by Numbers"
date:   2018-10-19
---

This is a Kubernetes tutorial meant to be used as an accompanying resource for a workshop. If you're looking for a more guided tutorial, I recommend going through the [Kubernetes Bootcamp](https://kubernetesbootcamp.github.io/kubernetes-bootcamp/index.html)

# 1. Setting up a local Kubernetes cluster

In order to start experimenting with Kubernetes, you'll need to have access to a cluster of computers with Kubernetes installed. If you don't have a spare datacenter laying around, you have several options. You could choose one of the major cloud provider container engines, Google Kubernetes Engine (GKE), Azure Container Service Engine (ACSE) or Amazon Elastic Container Service (ECS). These will provide you with an already setup Kubernetes cluster, but you'll be billed on your usage (which may or may not be free for you).

Instead, you could setup a cluster locally on your computer through the magic of virtual machines! The most popular options are:

* [minikube](https://github.com/kubernetes/minikube)
* [Docker For Mac](https://docs.docker.com/docker-for-mac/kubernetes/)
* [kubeadm-dind-cluster](https://github.com/kubernetes-sigs/kubeadm-dind-cluster)

I've had varying experiences with each of these. If you find one isn't working on your computer, I recommend just trying a different one rather than debugging the problem. Trust me on this one

I'll leave setting this up to you, if you manage to get it working then you have enough skill and dedication to get through this tutorial.

# 2. Starting a Pod

*pod.yaml*
{% highlight yaml %}
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
spec:
  restartPolicy: Never
  containers:
  - name: hello
    image: "ubuntu:latest"
    command: ["/bin/echo", "hello", "world"]
{% endhighlight %}

Start the pod:
<code class="inline-highlight">kubectl apply -f pod.yaml</code>

View the pod:
{% highlight text %}
$ kubectl get pods
NAME                         READY     STATUS      RESTARTS   AGE
hello-world                  0/1       Completed   0          47s
{% endhighlight %}

Delete the pod:
<code class="inline-highlight">kubectl delete pod hello-world</code>

And it's gone, never to return on its own:
{% highlight text %}
$ kubectl get pods
NAME                         READY     STATUS    RESTARTS   AGE
{% endhighlight %}

# 3. Deploying an application

*deployment.yaml*
{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
{% endhighlight %}

Deploy the application:
<code class="inline-highlight">kubectl apply -f deployment.yaml</code>

See the deployment state:
{% highlight text %}
kubectl get deployment nginx-deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE  AGE
nginx-deployment   2         2         2            0          2s
{% endhighlight %}

See the number of pods running:
{% highlight text %}
$ kubectl get pods
NAME                               READY  STATUS    RESTARTS  AGE
nginx-deployment-c4747d96c-2xdrm   1/1    Running   0         6s
nginx-deployment-c4747d96c-lh4kn   1/1    Running   0         6m
{% endhighlight %}

See one of the pod IP addresses:
{% highlight text %}
$ kubectl describe pod nginx-deployment-c4747d96c-2xdrm
Name:           nginx-deployment-c4747d96c-2xdrm
...
IP:             10.1.33.237
{% endhighlight %}

Delete the pod:
<code class="inline-highlight">kubectl delete pod nginx-deployment-c4747d96c-2xdrm</code>

And another one has come to take it's place:
{% highlight text %}
$ kubectl get pods
NAME                               READY  STATUS    RESTARTS  AGE
nginx-deployment-c4747d96c-g4vcp   1/1    Running   0         14s
nginx-deployment-c4747d96c-lh4kn   1/1    Running   0         7m
{% endhighlight %}

With a different IP address:
{% highlight text %}
$ kubectl describe pod nginx-deployment-c4747d96c-g4vcp
Name:           nginx-deployment-c4747d96c-g4vcp
IP:             10.1.33.238
{% endhighlight %}

We can update to the latest image:
<code class="inline-highlight">kubectl set image deployment/nginx-deployment nginx=nginx:1.15.5 --record</code>

{% highlight text %}
$ kubectl get pods
NAME                                READY  STATUS    RESTARTS  AGE
nginx-deployment-6f7f6746fd-6jbpq   1/1    Running   0         1m
nginx-deployment-6f7f6746fd-hqndt   1/1    Running   0         26s
{% endhighlight %}

{% highlight text %}
$ kubectl describe pods nginx-deployment-6f7f6746fd-m4x5c
Name:           nginx-deployment-6f7f6746fd-m4x5c
Containers:
  nginx:
    Image:          nginx:1.15.5
{% endhighlight %}

Scale up the number of pods:
<code class="inline-highlight">kubectl scale deployment nginx-deployment --replicas=3</code>

{% highlight text %}
$ kubectl get deployment nginx-deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE  AGE
nginx-deployment   3         3         3            3          14m
{% endhighlight %}

{% highlight text %}
$ kubectl get pods
NAME                                READY  STATUS    RESTARTS  AGE
nginx-deployment-6f7f6746fd-2q8d5   1/1    Running   0         1m
nginx-deployment-6f7f6746fd-5cb8s   1/1    Running   0         1m
nginx-deployment-6f7f6746fd-67mxl   1/1    Running   0         1m
{% endhighlight %}

# 4. Attaching it to a service

*service.yaml*
{% highlight yaml %}
kind: Service
apiVersion: v1
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - name: main
      protocol: TCP
      port: 80
      targetPort: 80
{% endhighlight %}

Create the service:
<code class="inline-highlight">kubectl apply -f service.yaml</code>

View your service:
{% highlight text %}
$ kubectl get service
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)  AGE
kubernetes   ClusterIP  10.96.0.1       <none>       443/TCP  36d
nginx        ClusterIP  10.108.11.129   <none>       80/TCP   37m
{% endhighlight %}

# 5. Setup an Ingress

*Sidenote*: If you're using Docker for Mac Kubernetes, you will need to install an ingress controller.

Create the namespace the controller will run in:
<code class="inline-highlight">kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml</code>

Start the pods:
<code class="inline-highlight">kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml</code>

Wait until pods are running:
<code class="inline-highlight">kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch</code>

*ingress.yaml*
{% highlight yaml %}
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx
  annotations:
    http.port: "443"
spec:
  backend:
    serviceName: nginx
    servicePort: 80
{% endhighlight %}

Create the ingress:
<code class="inline-highlight">kubectl apply -f ingress.yaml</code>

See the ingress with an external address:
{% highlight text %}
$ kubectl get ingress
NAME      HOSTS     ADDRESS     PORTS     AGE
nginx     *         localhost   80        49s
{% endhighlight %}

Connect to your application: [https://localhost](https://localhost)
