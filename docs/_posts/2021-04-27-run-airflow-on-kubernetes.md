---
title:  "Run Airflow on Kubernetes"
date:   2021-04-27 16:32:02 +0530
categories: tech airflow
---

# TL;DR:  
```
helm repo add airflow-stable https://airflow-helm.github.io/charts
helm repo update
helm install airflow airflow-stable/airflow
```


**Sections:**
1. [Why to run Airflow on Kubernetes](#why-to-run-airflow-on-kubernetes)
2. [Native support for Airflow to run Kubernetes](#native-support-for-airflow-to-run-kubernetes)
3. [Different ways to install Airflow on Kubernetes](#different-ways-to-install-airflow-on-kubernetes)
4. [Install it the recommended way](#install-airflow-on-kubernetes-using-the-recommended-way-helm-chart)
5. [Production grade Airflow installation](#production-grade-airflow-installation)


##  Why to run Airflow on Kubernetes
When you run Airflow on Kubernetes, you also normally configure the Executor as Kubernetes. This means all your jobs run as separate docker container (pods in Kubernetes). This gives your jobs complete freedom to used all the resources assigned to your pod. Airflow just has to ask Kubernetes to run a new docker image that has the code you want to run as your task. If you don't use this executor then it actually makes no sense to run the Airflow on K8s since it will not be utilizing the Kubernetes for launching different jobs.   

The real benefit comes when you start using Kubernetes Executor with Kubernetes Operator. Running jobs as docker containers itself provides you numerous advantages over running different jobs on the same machine. To give you an example, imagine two different python jobs needing different versions of the same library. If the jobs are on docker, you can basically create two python images with different version of this library and each of those can run without interfering with each other. Same is obviously not true when running these on the same machine and at the same time.  

## Native support for Airflow to run Kubernetes

Airflow officially supports running as a docker container and its image can be found [here](https://hub.docker.com/r/apache/airflow). There is an official helm chart which you can follow to install Airflow on Kubernetes. If you get stuck somewhere, do come back here :)


## Different ways to install Airflow on Kubernetes

Airflow has a docker image you can use to create your own Deployments, Service, Ingress etc. But doing all the manually to install Airflow might become complex very fast because there will be plenty of options you will have to deal with. We will not go by that route.  
The other option is to install using a helm chart which creates all the Kubernetes yaml specs for you and you only have to deal with a single file of parameters (`values.yaml`)


## Install Airflow on Kubernetes using the recommended way (Helm chart)

Make sure you have [helm](https://helm.sh/docs/intro/install/) installed.  

Then you need to add the Airflow helm repo in your local helm.  
```
helm repo add airflow-stable https://airflow-helm.github.io/charts
helm repo update
```

One that is done, confirm that you are able to access your desired Kubernetes cluster with `klubectl`  
```
kubectl get ns
```

Now you can create a namespace on your choice where you would like to install Airflow. Let's take `dev-airflow` as an example namespace.  
```
kubectl create ns dev-airflow
```

We can now install the Airflow helm chart:
```
helm install airflow airflow-stable/airflow --namespace dev-airflow
```
It might take a few minutes to install so just be patient. Once it is done, you will be presented with some text which will help you access the airflow installed on your browser. It will look something like this:
```
Congratulations, you have just deployed Apache Airflow!

----------------
Access the Webserver Service with your browser
----------------
  export POD_NAME=$(kubectl get pods --namespace dev-airflow -l "component=web,app=airflow" -o jsonpath="{.items[0].metadata.name}")
  echo "URL: http://127.0.0.1:8080"
  kubectl port-forward --namespace dev-airflow $POD_NAME 8080:8080
```
Using the information above you can open Airflow on browser and put `admin/admin` as the `username/password`.

If you are able to login then congratulations, you have successfully installed Airflow on Kubernetes.  
If you just wanted to do a POC around the installation then you are done. But if you are trying to do a serious deployment then this will not suffice. Here are some of the questions you must be asking to yourself now:
1. Where will I add the Airflow DAGs?
2. Where will the Airflow store the metadata about the DAGs?
3. Where will the Airflow store the logs of my DAGs?
If your question is not here then you might it [here](https://github.com/airflow-helm/charts/tree/main/charts/airflow#airflow-configs)

To deal with all of the questions above, you will have to create and manage a file called `values.yaml` which contains all the different configuration to control your Airflow instance installation. Just download the file from [here](https://github.com/airflow-helm/charts/blob/main/charts/airflow/values.yaml), change the configurations you want to control (based on the [questions](https://github.com/airflow-helm/charts/tree/main/charts/airflow#airflow-configs) you have) and then re-install the helm chart by:
```
helm uninstall airflow -n dev-airflow
helm install airflow -f values.yaml airflow-stable/airflow --namespace dev-airflow
```
Note that the last time we installed the chart we did not provide any values.yaml file. At that time it used the default values present [here](https://github.com/airflow-helm/charts/blob/main/charts/airflow/values.yaml). There are just too many ways you can customize this installation and it is really not possible to cover that here in this blog. I would recommend going through the question list above and modify the `values.yaml` according to the answers. And then of course re-install the chart. 

## Production grade Airflow installation

To run a production grade you will have keep the following points mind.  
1. **Configure an external database in Airflow instead of using the default container based Postgres instance.**  
  To modify this you will have to edit the `values.yaml` file and modify the following parameters:
    ```
      postgresql:
        enabled: false

      externalDatabase:
        type: postgres
        host: postgres.example.org
        port: 5432
        database: airflow_cluster1
        user: airflow_cluster1
        passwordSecret: "airflow-cluster1-postgres-password"
        passwordSecretKey: "postgresql-password"

        # use this for any extra connection-string settings, e.g. ?sslmode=disable
        properties: ""
    ```
    Yes, these are the same options as provide [here](https://github.com/airflow-helm/charts/tree/main/charts/airflow#how-to-use-an-external-database-recommended). So in the coming recommendations, I will not mention the exact props, but instead give you the link to the configuration.

2. **Configure Github-sync for storing and managing DAGs.**  
  When you deploy Airflow in your local machine to test it out, you can just put the DAG files in a folder and Airflow stars picking them up. But when the Airflow is installed on K8s, image how you provide DAGs to it (As of now, there is no option on Airflow UI to submit a new DAG). You can create custom image of Airflow which already has DAGs but then you will have to create a new image every time you change something in a DAG or add a new one. You can mount a PVC having DAGs. But adding DAGs to the PVC is again something which you will have to figure out.  
  The best way is to provide Airflow a Github repository (with read creds) and let it poll for any changes in a DAG or addition of a new DAG. Github is something all your developers are familiar with and anyone who wants to add a DAG to the Airflow can add it just by submitting a PR to the Github instead of knowing the K8s interface in order to use the other methods of adding DAGs. Having the DAGs on Github also allows you to control who is submitting what to the Airflow and complete history of change in your DAGs. The configurations to do so can be found [here](https://github.com/airflow-helm/charts/tree/main/charts/airflow#how-to-use-an-external-database-recommended)
3. **Enable logs persistence.**  
  When an Airflow DAG runs on K8s, it generates some logs. These are nothing but the pods logs which are created as part of the job execution. By default, Airflow will pull the logs from the pod itself. So as long as your pod is there on the K8s (in any state like Failed or Completed), you will be able to see those logs on Airflow UI. But as soon you remove the pods (Maybe to cleanup the completed pods), the logs for that task will be gone. To fix this, Airflow can be configured to push the logs to an external system so that it can show you the logs even if the corresponding pod is deleted. You can configure it to store on storages like S3. You can find the configurations options [here](https://github.com/airflow-helm/charts/tree/main/charts/airflow#option-2---remote-bucket-recommended)
4. **Integrate with and external authentication system.**  
  By default Airflow will create static user/passwords for users which are stored within the Airflow database. In a production environment, you will probably required some kind of LDAP/Openid authentication. This is something that is not supported by the helm chart but it is supported by the Airflow image itself. Maybe I will write a blog about that.
