---
layout: post
title:  "Airflow on Kubernetes and HA on API Server"
date:   2021-10-25 16:32:02 +0530
categories: tech airflow
---


## This article is for:
    - If you are running Airflow on Kubernetes with Kubernetes executor and
    - You are running the K8s API server on HA and
    - You are wondering if your Airflow using the K8s API server's HA


# TL;DR:

If your Airflow is running with the same Kuberneters cluster where the Airflow jobs run, chances are your Airflow is already using your highly available Kubernetes API server. No need to worry.

## What are you going to read about:
    I had this situation at my work where quite a few of our Airflow jobs would suddenly fail with an error thrown by Airflow saying it can't connect to the API server. We figured out that it was the heavy load put by a huge number of Airflow jobs getting started at the same time. At that time, we were neither doing HA on the Kubernetes master nor we had tainted the master node to not schedule anything on it (I know... It was a new cluster... so...). So the Airflow job would schedule a lot of pods on the master node, which would cause the master node itself to go down, hence the Airflow would not be able to find the status of the pods from the API server and eventually mark them as failed (though they might have completed).
    So to fix all of this, we added two more master nodes to have HA on the API server (and other services as well). We tained all the master nodes so that none of the workload (including Airflow jobs) would get scheduled there. Airflow is still putting load on the worker nodes which is causing the node outage but at least the API servers are safe. But that's not the problem we are going to discuss.
    Setting up all of these had us wonder _is Airflow actually using the HA enabled K8s API server?_. If it is not , should we put a Load Balancer on top of the API servers and ask Airflow to use that?

    To answer this question, we had to answer a few other questions

## Which K8s API Server is the Airflow using?
    I checked Airflow [docs](https://airflow.apache.org/docs/apache-airflow/2.1.1/executor/kubernetes.html#kubernetesexecutor-architecture) to confirm that the Airflow actually uses the API server to fetch pod status, logs, etc. It can be confirmed by looking at the [code](https://github.com/apache/airflow/blob/a71f9d6f25e3255ac33755da2590c398062db9d1/airflow/providers/cncf/kubernetes/utils/pod_launcher.py#L114) as well. Airflow initializes this library by calling a method called `load_incluster_config()`. This is used when the person using the library in their code knows that the core will be used inside a Kubernetes cluster. In short, if you run the library inside the Kubernetes cluster in a POD, just by calling `load_incluster_config`, you can init the lib and it cal start getting status of pods (or anything else)

## How is the library magically getting initialized?
    The library to work, has to know the API server URL, have the credentials to query that API server. Where is the library getting all this?
    It gets this by the available environment variables and volume mount on the pod. Whenever Kubernetes starts a POD, it by default sends the API server info as environment variables. For example the `KUBERNETES_SERVICE_HOST` and `KUBERNETES_SERVICE_PORT`. Along with this, the K8s also mounts a `default` secret in the namespace inside the container. Using the combination of these two, the library can do whatever the `default` secret is allowed to do.
    When I checked the value of `KUBERNETES_SERVICE_HOST`, I first thought it would be an IP of our K8s API servers. Upon asking the infra guys, I found out that it wasn't.

## What is this strange IP assigned to `KUBERNETES_SERVICE_HOST`?
    At first I was clueless on this. But after pondering on this a little bit, I remembered that the K8s exposes the API server as a K8s [Service](https://kubernetes.io/docs/concepts/services-networking/service/). When I checked the IP of the service by doing `kubectl get svc/kubernetes -n default`, I found out that this IP was the same as the value of `KUBERNETES_SERVICE_HOST`.


## Now I know that Airflow is using this Service, but is this load balanced?
    Normally, we use a K8s service to load balance multiple pods of an app. The way a `Service` determines which pods to serve is by looking at the label selectors it has defined with (like `app=mywebapp`). So I looked for a selector by doing `kubectl get svc/kubernetes -n default -o wide`, and there were no selectors? It made me wonder if the Services would exist with any selectors? Turns out they [can](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors). Looking at this I thought maybe we had defined a `Endpoints` which is linked to the service. I did `kubectl get endpoints/kubernetes -o yaml`, and boom! All the three HA URLs for our K8s API servers were there.

## Conclusion
    Hence I was able to conclude that the Airflow is always taking advantage of your HA on your master k8s master nodes. And no matter how many master nodes you add, they will automatically get added to the `Endpoints` and services within k8s can enjoy the HA. To summarize, here is how it looks:
    ```
    Airflow scheduler pod -> gets the `KUBERNETES_SERVICE_HOST` env variable -> It points to the 'kubernetes' service in the default namespace -> service points to HA endpoints of your k8s master.
    ```

