
# CI/CD Pipeline on Kubernetes in 5 Minutes

**After 3 commands a CI/CD pipeline with Jenkins, SonarQube, Nexus and a Docker Registry will be up and running on your Kubernetes cluster.**

Follow this to **[install a microk8s kubernetes cluster](https://www.devopswiki.co.uk/kubernetes/microk8s-install)** on your laptop and optionally other nodes.


---


## Step 1 | Run Jenkins on Kubernetes

These are the **3 commands** to run both the Jenkins master and the workers on a (possibly) distributed Kubernetes cluster.

```
git clone https://github.com/devops4me/docker-jenkins-cluster.git
cd docker-jenkins-cluster
kubectl apply -f jenkins-deployment.yaml
```

That's it. Your Kubernetes orchestrated Jenkins cluster is ready.


---


## Step 2 | View Jenkins jobs running in Kubernetes

The sample jobs include
- a JAVA microservice and
- a JAVA library

```
kubectl get services -o wide
kubectl -n default logs -f deployment/jenkins --all-containers=true --since=5m
watch kubectl get pods -o wide
```

Visit the Jenkins UI with the IP address against the Jenkins service. Go to the jobs, click **`Build Now`** and then watch both the logs and the pods showing Kubernetes running dockerized pipelines on one or more nodes.


---


## Appendix | Kubernetes Jenkins Plugin

To view or change how the Kubernetes plugin configuration go to Jenkins | Manage Jenkins | Configure System and scroll down to the clouds section. The default configuration assumes

- the kubernetes API url is **`https://kubernetes.default.svc`** from inside the cluster
- the Jenkins master url the slaves connect to is **`http://jenkins`** as **jenkins** is the service name
- the Jenkins master and slaves pods all run in the **`default`** Kubernetes namespace

This **[sample Java microservice](https://github.com/apolloakora/bank-account)** shows how the **Jenkinsfile** and the **pod-template yaml** are written to fascilitate execution on a Kubernetes platform.
