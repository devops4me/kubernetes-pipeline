
# CI/CD Pipeline on Kubernetes in 5 Minutes

**After 3 commands a CI/CD pipeline with Jenkins, SonarQube, Nexus and a Docker Registry will be up and running on your Kubernetes cluster.**

Follow this to **[install a microk8s kubernetes cluster](https://www.devopswiki.co.uk/kubernetes/microk8s-install)** on your laptop and optionally other nodes.


---


## Step 1 | Run Jenkins on Kubernetes

These are the **3 commands** to run both the Jenkins master and the workers on a (possibly) distributed Kubernetes cluster.

```
git clone https://github.com/devops4me/docker-jenkins.git
cd docker-jenkins
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

## Step 3 | Create Dockerhub Credentials Kubernetes Secret

The dockerhub credentials will be used by **[Google Kaniko](https://www.devopswiki.co.uk/kubernetes/kaniko)** to push freshly built docker images into the a docker registry.

```
docker login
cat ~/.docker/config.json
kubectl create secret generic registrycreds \
    --from-file=config.json=$HOME/.docker/config.json
docker logout
```

### Docker's config.json file

If using Dockerhub the config.json file will look something like this.

```json
{
	"auths": {
		"https://index.docker.io/v1/": {
			"auth": "AzByCxxxxxxxxxxxxxxxxXcYbZa"
		}
	},
	"HttpHeaders": {
		"User-Agent": "Docker-Client/18.09.7 (linux)"
	}
}
```


---


## Step 4 | Create [safedb.net](https://github.com/devops4me/safedb.net) Git Credentials Kubernetes Secret

The **[safedb.net github credentials](https://github.com/devops4me/safedb.net)** will be used by the ruby release gem instigated by the **[kubernetes Jenkins release pod](https://github.com/devops4me/safedb.net/blob/master/pod-image-release.yaml)** to push the bumped up version number into the Git repository.

Go to a scratch folder on the machine and run these commands.
```
safe login <<book>>
safe open <<chapter>> <<verse>>
safe eject github.ssh.config
safe eject safedb.code.private.key
chmod 600 safedb.code.private.key.pem
kubectl create secret generic safedbgitsshconfig \
    --from-file=config=$PWD/github.ssh.config
rm github.ssh.config
kubectl create secret generic safedbgitsshkey \
    --from-file=safedb.code.private.key.pem=$PWD/safedb.code.private.key.pem
rm safedb.code.private.key.pem
```


---


## Step 5 | Create [safedb.net RubyGem Credentials](https://rubygems.org/gems/safedb) Kubernetes Secret

The **[safedb.net rubygem.org credentials](https://rubygems.org/gems/safedb)** will be used by the ruby release gem instigated by the **[kubernetes Jenkins release pod](https://github.com/devops4me/safedb.net/blob/master/pod-image-release.yaml)** to deploy the packaged gem to the public gem repository.

```
docker login
cat ~/.docker/config.json
kubectl create secret generic registrycreds \
    --from-file=config.json=$HOME/.docker/config.json
docker logout
```

### Docker's config.json file

If using Dockerhub the config.json file will look something like this.

```json
{
	"auths": {
		"https://index.docker.io/v1/": {
			"auth": "AzByCxxxxxxxxxxxxxxxxXcYbZa"
		}
	},
	"HttpHeaders": {
		"User-Agent": "Docker-Client/18.09.7 (linux)"
	}
}
```





## Appendix | Kubernetes Jenkins Plugin

To view or change how the Kubernetes plugin configuration go to Jenkins | Manage Jenkins | Configure System and scroll down to the clouds section. The default configuration assumes

- the kubernetes API url is **`https://kubernetes.default.svc`** from inside the cluster
- the Jenkins master url the slaves connect to is **`http://jenkins`** as **jenkins** is the service name
- the Jenkins master and slaves pods all run in the **`default`** Kubernetes namespace

This **[sample Java microservice](https://github.com/apolloakora/bank-account)** shows how the **Jenkinsfile** and the **pod-template yaml** are written to fascilitate execution on a Kubernetes platform.
