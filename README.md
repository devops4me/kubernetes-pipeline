
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
safe open github devops4me
safe write github.ssh.config # remove any extraneous config sections
safe write safedb.code.private.key
chmod 600 safedb.code.private.key.pem
kubectl create secret generic safegitsshconfig \
    --from-file=config=$PWD/config
rm config
kubectl create secret generic safegitsshkey \
    --from-file=safedb.code.private.key.pem=$PWD/safedb.code.private.key.pem
rm safedb.code.private.key.pem
kubectl get secret safegitsshconfig --output=yaml
kubectl get secret safegitsshkey --output=yaml
```

## Step 5 | Check the Pod Yaml Definitions

This is an example of a Pod's Yaml definition that uses the **`.ssh/config`** and the **`ssh private key`** created above.

```
metadata:
    labels:
        pod-type: jenkins-worker
spec:
    containers:
    -   name: jnlp
        env:
        -   name: CONTAINER_ENV_VAR
            value: jnlp
    -   name: safehaven
        image: devops4me/haven:latest
        imagePullPolicy: Always
        volumeMounts:
        -   name: gitsshconfig
            mountPath: /root/gitsshconfig
        -   name: gitsshkey
            mountPath: /root/gitsshkey
        command:
        -   cat
        tty: true
        env:
        -   name: CONTAINER_ENV_VAR
            value: safehaven
    volumes:
        -   name: gitsshconfig
            secret:
                secretName: safegitsshconfig
        -   name: gitsshkey
            secret:
                secretName: safegitsshkey
                defaultMode: 256
```

### Important Points

Note that

- we do not use **`~`** because kubernetes does not understand it
- we do not map to **`/root/.ssh`** as this folder will be read only and known hosts writing will fail
- in the Jenkinsfile we copy **`/root/gitsshconfig/config`** to the expected **`/root/.ssh/config`**
- inside the config file we have specified the **`/root/gitsshkey/<<private-key-name.pem>>`** location
- we use **`defaultMode: 256`** to set the **`0400`** file permissions for the key


## Step 6 | Check the Jenkinsfile Definitions

Within the Jenkinsfile we need to take care of a number of issues.

```
        stage('Release to RubyGems.org')
        {
            agent
            {
                kubernetes
                {
                    yamlFile 'pod-image-release.yaml'
                }
            }
            when {  environment name: 'GIT_BRANCH', value: 'origin/master' }
            steps
            {
                container('safehaven')
                {
                    checkout scm
                    sh 'mkdir -p $HOME/.ssh && cp $HOME/gitsshconfig/config $HOME/.ssh/config'
                    sh 'git config --global user.email apolloakora@gmail.com'
                    sh 'git config --global user.name "Apollo Akora"'
                    sh 'ssh -i $HOME/gitsshkey/safedb.code.private.key.pem -vT git@safedb.code || true'
                    sh 'git remote set-url --push origin git@safedb.code:devops4me/safedb.net.git'
                    sh 'gem bump minor --tag --push --release --file=$PWD/lib/version.rb'
                }
            }
        }
```


### Verify the Git SSH Credentials

We verify the git push credentials within the **[safedb.net Jenkinsfile](https://github.com/devops4me/safedb.net/blob/master/Jenkinsfile)** in the rubygem deploy stage right before the **`gem bump --release`** command.

#### ssh -i ~/.ssh/safedb.code.private.key.pem -vT git@safedb.code

If the **ssh to github** verification works - we can be confident that the kubernetes secrets have been correctly configured.

### Set the Git Push Upstream Url

The **[safedb.net Jenkinsfile](https://github.com/devops4me/safedb.net/blob/master/Jenkinsfile)** must configure the **remote url for git push** just before the git push is done.

#### git remote set-url --push origin git@safedb.code:devops4me/safedb.net.git

### Running Without Credentials

It is important for Jenkins jobs to complete safely without credentials as long as they are not in a given branch (usually master). This allows the team to run the jobs locally from development branches without a deployment happening.

### ssh command exits with status 1

Even when the ssh trial login command succeeds it still exits with return code 1 so we put **`|| true`** to prevent it failing the build.


---


## Step 7 | Create [safedb.net RubyGem Credentials](https://rubygems.org/gems/safedb) Kubernetes Secret

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
