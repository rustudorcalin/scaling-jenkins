## Introduction

[Jenkins](https://jenkins.io) is an open-source continuous integration and continuous delivery tool, which can be used to automate building, testing and deploying software. It is widely considered the most popular automation server, being used by more than 1,000,000 users worldwide. Advantages of Jenkins include:
- open source, with huge community support
- Java based, making it portable to all major platforms
- Rich plugin ecosystem (more than 1000) 

It provides support for all popular Source Control Management systems (Git, SVN, Mercurial and CVS), popular build tools (Ant, Maven, Grunt), shell scripts or Windows batch commands, as well as testing frameworks and report generators. Jenkins plugins provide support (among many others) for technologies like **Docker** and **Kubernetes**, which enable the creation and deployment of cloud-based microservice environments, both for testing as well as production deployments. 

Jenkins supports the master-slave architecture (many slaves/nodes/build servers work for a master) making it highly **scalable**. The master's job is to handle scheduling build jobs, distributing the jobs to slaves for actual execution, monitor the slaves and get the build results. It can also execute build job directly. The job of the slaves is to build the job distributed by the master. A job can be configured to run on a particular type of slave, or if no specifications are needed you can let Jenkins pick the next available slave.

Jenkins scalability gives you lots of benefits:
- ability to run many build plans in parallel
- automatically spinning up and removing slaves based on need, which saves costs
- distributing the load

Even if the scalability feature is available in Jenkins out-of-the-box, the process of making Jenkins scalable is not straightforward. There are plenty of options available to implement Jenkins scaling. One of the powerful options available right now is Jenkins scaling on top of Kubernetes.

## What is Kubernetes?

Kubernetes is an open source containers orchestration tool, managing containers on which applications run. Its main purpose is to manage containerized applications in a cluster of nodes and to facilitate their deployment, scaling, updating, maintenance or service discovery. More details about what Kubernetes is, what it can do, you can find out from the official documentation [page](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes).

Kubernetes is one of the best tools for managing scalable solutions. Almost every application can be containerized, Jenkins too, and that’s why it makes Kubernetes a very nice option to use.

![01](images/01-rancher-jenkins-master-slave-architecture.png)

## Hands on

Before starting, let's take a moment and describe what we want to accomplish. We want to create a Kubernetes cluster and on top to deploy Jenkins Master. We will use `kubernetes` plugin (currently version [1.13.0](https://github.com/jenkinsci/kubernetes-plugin)) to implement Jenkins scaling on top of this cluster by running dynamic agents. The plugin will create a Kubernetes Pod for each agent started defined by the Docker image to run, and will stop it after each build. Agents will be launched using JNLP, so it is expected that the image will connect automatically to the Jenkins master. 

**Prerequisites and Setup**
- GCP account, free tier is enough
- Kubernetes cluster, this demo uses GCP's GKE but any other [provider](https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/hosted-kubernetes-clusters/) works the same
- Linux box where Rancher will be running (also will be used to build custom Jenkins images)
- docker hub account, to push these custom images (master and slave)

Let's start by building our own custom images and then push them to [hub.docker.com](https://hub.docker.com). We will use for this docker, which can be installed following this [tutorial](https://docs.docker.com/install/linux/docker-ce/centos/).

Once installed we need to prepare our [Dockerfiles](https://docs.docker.com/engine/reference/builder/).

```
[root@rancher-instance jenkins-kubernetes]# ll
total 16
-rw-r--r--. 1 root root  265 Oct 21 12:58 Dockerfile-jenkins-master
-rw-r--r--. 1 root root  322 Oct 21 13:16 Dockerfile-jenkins-slave-jnlp1
-rw-r--r--. 1 root root  315 Oct 21 13:05 Dockerfile-jenkins-slave-jnlp2
-rw-r--r--. 1 root root    0 Oct 21 13:15 empty-test-file
-rwxr-xr-x. 1 root root 3750 Oct 21 08:08 jenkins-slave
```

```
[root@rancher-instance jenkins-kubernetes]# cat Dockerfile-jenkins-master

FROM jenkins/jenkins:lts

# Plugins for better UX (not mandatory)
RUN /usr/local/bin/install-plugins.sh ansicolor
RUN /usr/local/bin/install-plugins.sh greenballs

# Plugin for scaling jenkins slaves
RUN /usr/local/bin/install-plugins.sh kubernetes

USER jenkins
```

```
[root@rancher-instance jenkins-kubernetes]# cat Dockerfile-jenkins-slave-jnlp1

FROM jenkins/jnlp-slave

COPY jenkins-slave /usr/local/bin/jenkins-slave

# For testing purpose only
COPY empty-test-file /usr/local/bin/jenkins-slave1

ENTRYPOINT ["jenkins-slave"]
```

```
[root@rancher-instance jenkins-kubernetes]# cat Dockerfile-jenkins-slave-jnlp2

FROM jenkins/jnlp-slave

COPY jenkins-slave /usr/local/bin/jenkins-slave

# For testing purpose only
COPY empty-test-file /usr/local/bin/jenkins-slave2

ENTRYPOINT ["jenkins-slave"]
```

OBS1: `jenkins-slave` script can be downloaded from the official GitHub repo of [jenkinsci/docker-jnlp-slave](https://github.com/jenkinsci/docker-jnlp-slave).

OBS2: `empty-test-file`, file needed to be present in same folder as docker files, will be used for testing; for first Docker Slave JNLP image will be copied as `jenkins-slave1`, for second one as `jenkins-slave2`.

Once these files are ready we can build and push the images to hub.

```bash
[root@rancher-instance jenkins-kubernetes]# docker build -f Dockerfile-jenkins-master -t jenkins-master .
```

<details><summary>Click for full command output</summary>

```bash
Sending build context to Docker daemon 12.29 kB
Step 1/5 : FROM jenkins/jenkins:lts
Trying to pull repository docker.io/jenkins/jenkins ... 
lts: Pulling from docker.io/jenkins/jenkins
05d1a5232b46: Pull complete 
5cee356eda6b: Pull complete 
89d3385f0fd3: Pull complete 
80ae6b477848: Pull complete 
40624ba8b77e: Pull complete 
8081dc39373d: Pull complete 
8a4b3841871b: Pull complete 
b919b8fd1620: Pull complete 
2760538fe600: Pull complete 
bcb851da81db: Pull complete 
eacbf73f87b6: Pull complete 
9a7e396a0cbd: Pull complete 
8900cde5602e: Pull complete 
c8f62fde3f4d: Pull complete 
eb91939ba069: Pull complete 
b894a41fcbe2: Pull complete 
b3c60e932390: Pull complete 
18f663576636: Pull complete 
4445e4b557b3: Pull complete 
f09e9b4be8ed: Pull complete 
e3abe5324295: Pull complete 
432eff1ecbb4: Pull complete 
Digest: sha256:d5c835407130a393becac222b979b120c675f8cd815fadd085adb76b216e4ce1
Status: Downloaded newer image for docker.io/jenkins/jenkins:lts
 ---> 9cff19ad8c8b
Step 2/5 : RUN /usr/local/bin/install-plugins.sh ansicolor
 ---> Running in ff752eeb107d

Creating initial locks...
Analyzing war...
Registering preinstalled plugins...
Using version-specific update center: https://updates.jenkins.io/2.138...
Downloading plugins...
Downloading plugin: ansicolor from https://updates.jenkins.io/2.138/latest/ansicolor.hpi
 > ansicolor depends on workflow-step-api:2.12;resolution:=optional
Skipping optional dependency workflow-step-api

WAR bundled plugins:


Installed plugins:
ansicolor:0.5.2
Cleaning up locks
 ---> a018ec9e38e6
Removing intermediate container ff752eeb107d
Step 3/5 : RUN /usr/local/bin/install-plugins.sh greenballs
 ---> Running in 3505e21268b2

Creating initial locks...
Analyzing war...
Registering preinstalled plugins...
Using version-specific update center: https://updates.jenkins.io/2.138...
Downloading plugins...
Downloading plugin: greenballs from https://updates.jenkins.io/2.138/latest/greenballs.hpi

WAR bundled plugins:


Installed plugins:
ansicolor:0.5.2
greenballs:1.15
Cleaning up locks
 ---> 0af36c7afa67
Removing intermediate container 3505e21268b2
Step 4/5 : RUN /usr/local/bin/install-plugins.sh kubernetes
 ---> Running in ed0afae3ac94

Creating initial locks...
Analyzing war...
Registering preinstalled plugins...
Using version-specific update center: https://updates.jenkins.io/2.138...
Downloading plugins...
Downloading plugin: kubernetes from https://updates.jenkins.io/2.138/latest/kubernetes.hpi
 > kubernetes depends on workflow-step-api:2.14,apache-httpcomponents-client-4-api:4.5.3-2.0,cloudbees-folder:5.18,durable-task:1.16,jackson2-api:2.7.3,variant:1.0,kubernetes-credentials:0.3.0,pipeline-model-extensions:1.3.1;resolution:=optional
Downloading plugin: workflow-step-api from https://updates.jenkins.io/2.138/latest/workflow-step-api.hpi
Downloading plugin: apache-httpcomponents-client-4-api from https://updates.jenkins.io/2.138/latest/apache-httpcomponents-client-4-api.hpi
Downloading plugin: cloudbees-folder from https://updates.jenkins.io/2.138/latest/cloudbees-folder.hpi
Downloading plugin: durable-task from https://updates.jenkins.io/2.138/latest/durable-task.hpi
Downloading plugin: jackson2-api from https://updates.jenkins.io/2.138/latest/jackson2-api.hpi
Downloading plugin: variant from https://updates.jenkins.io/2.138/latest/variant.hpi
Skipping optional dependency pipeline-model-extensions
Downloading plugin: kubernetes-credentials from https://updates.jenkins.io/2.138/latest/kubernetes-credentials.hpi
 > workflow-step-api depends on structs:1.5
Downloading plugin: structs from https://updates.jenkins.io/2.138/latest/structs.hpi
 > kubernetes-credentials depends on apache-httpcomponents-client-4-api:4.5.5-3.0,credentials:2.1.7,plain-credentials:1.3
Downloading plugin: credentials from https://updates.jenkins.io/2.138/latest/credentials.hpi
Downloading plugin: plain-credentials from https://updates.jenkins.io/2.138/latest/plain-credentials.hpi
 > cloudbees-folder depends on credentials:2.1.11;resolution:=optional
Skipping optional dependency credentials
 > plain-credentials depends on credentials:2.1.5
 > credentials depends on structs:1.7

WAR bundled plugins:


Installed plugins:
ansicolor:0.5.2
apache-httpcomponents-client-4-api:4.5.5-3.0
cloudbees-folder:6.6
credentials:2.1.18
durable-task:1.26
greenballs:1.15
jackson2-api:2.8.11.3
kubernetes-credentials:0.4.0
kubernetes:1.13.0
plain-credentials:1.4
structs:1.17
variant:1.1
workflow-step-api:2.16
Cleaning up locks
 ---> dd19890f3139
Removing intermediate container ed0afae3ac94
Step 5/5 : USER jenkins
 ---> Running in c1066861d5a3
 ---> 034e27e479c5
Removing intermediate container c1066861d5a3
Successfully built 034e27e479c5
```
</details></br>

Check the newly created image.

```
[root@rancher-instance jenkins-kubernetes]# docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
jenkins-master              latest              034e27e479c5        16 seconds ago      744 MB
docker.io/jenkins/jenkins   lts                 9cff19ad8c8b        10 days ago         730 MB
```

Login to docker using your own credentials.

```
[root@rancher-instance jenkins-kubernetes]# docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username (calinrus): 
Password: 
Login Succeeded
```

Tag the image you want to push, then push it.

```
[root@rancher-instance jenkins-kubernetes]# docker tag jenkins-master calinrus/jenkins-master
[root@rancher-instance jenkins-kubernetes]# docker push calinrus/jenkins-master
```

<details><summary>Click for full command output</summary>

```bash
The push refers to a repository [docker.io/calinrus/jenkins-master]
b267c63b5961: Pushed 
2cd1dc56ef56: Pushed 
e99d7d8d116f: Pushed 
8d117101392a: Mounted from jenkins/jenkins 
c2607b4e8ae4: Mounted from jenkins/jenkins 
81e4bc7cb1f1: Mounted from jenkins/jenkins 
8bac294d4ee8: Mounted from jenkins/jenkins 
707f669f3d58: Mounted from jenkins/jenkins 
ac2b51b56ac6: Mounted from jenkins/jenkins 
1b2b61bef21f: Mounted from jenkins/jenkins 
efe1c25100f5: Mounted from jenkins/jenkins 
8e656983ccf7: Mounted from jenkins/jenkins 
ba000aef226d: Mounted from jenkins/jenkins 
a046c3cdf994: Mounted from jenkins/jenkins 
67e27eb293e8: Mounted from jenkins/jenkins 
bdd1835d949d: Mounted from jenkins/jenkins 
84bbcb8ef932: Mounted from jenkins/jenkins 
0d67aa2185d5: Mounted from jenkins/jenkins 
3499b696191f: Pushed 
3b2a1688b8f3: Pushed 
b7c56a9790e6: Mounted from jenkins/jenkins 
ab016c9ea8f8: Mounted from jenkins/jenkins 
2eb1c9bfc5ea: Mounted from jenkins/jenkins 
0b703c74a09c: Mounted from jenkins/jenkins 
b28ef0b6fef8: Mounted from jenkins/jenkins 
latest: digest: sha256:6b2c8c63eccd795db5b633c70b03fe1b5fa9c4a3b68e3901b10dc3af7c3549f0 size: 5552
```
</details></br>

You will need to repeat same commands in order to build the two images for the Jenkins JNLP slaves.

```
docker build -f Dockerfile-jenkins-slave-jnlp1 -t jenkins-slave-jnlp1 .
docker tag jenkins-slave-jnlp1 calinrus/jenkins-slave-jnlp1
docker push calinrus/jenkins-slave-jnlp1

docker build -f Dockerfile-jenkins-slave-jnlp2 -t jenkins-slave-jnlp2 .
docker tag jenkins-slave-jnlp2 calinrus/jenkins-slave-jnlp2
docker push calinrus/jenkins-slave-jnlp2
```

If everything went fine in your docker hub account you should see something like this:

![02](images/02-rancher-docker-hub.png)


## Starting a Rancher 2.0 instance

Let's start now a Rancher 2.0 instance, that will help us deploy a GKE cluster. There is a very intuitive getting started guide for this purpose [here](https://rancher.com/quick-start/). Let’s create a hosted Kubernetes cluster and wait for it to turn `Active`. As soon as it's ready we can deploy Jenkins master (based on the image we just built) and create some services. If you are familiar with `kubectl` you can achieve this from command line, otherwise you can use Rancher's UI.

```
[root@rancher-instance k8s]# ll
total 8
-rw-r--r--. 1 root root 666 Oct 20 13:26 deployment.yml
-rw-r--r--. 1 root root 328 Oct 20 19:21 service.yml
```

```
[root@rancher-instance k8s]# cat deployment.yml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
        - name: jenkins
          image: calinrus/jenkins-master
          env:
            - name: JAVA_OPTS
              value: -Djenkins.install.runSetupWizard=false
          ports:
            - name: http-port
              containerPort: 8080
            - name: jnlp-port
              containerPort: 50000
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          emptyDir: {}
```          
          
```
[root@rancher-instance k8s]# cat service.yml 
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: jenkins

---

apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp
spec:
  type: ClusterIP
  ports:
    - port: 50000
      targetPort: 50000
  selector:
    app: jenkins
```


From Rancher, click on your managed cluster (called `jenkins` in this demo), go to Workloads tab, select the (local) file and hit Import. Wait for the Workload to finish.

![03](images/03-rancher-deploy-workloads-from-yaml.png)
![04](images/04-rancher-import.png)
![06](images/06-rancher-workload-progress-done.png)

We need a way to access Jenkins master (UI) instance. For this we need to create some services. One service will be created as a LoadBalancer, so we will get a public IP to access Jenkins from Internet, the other one will be a ClusterIP needed for internal communication between master and (future) slaves.
In `Load Balancing` tab follow same process, hit Import YAML, select the file containing our services and Import it.

![07](images/07-rancher-load-balancing.png)
![08](images/08-rancher-load-balancing-status.png)

Wait a bit for the public IP as it gets created. As soon as service turns `Active` you can find it by clicking on the `three vertical dots` at the right end of the page and select `View/Edit YAML`

![09](images/09-rancher-get-public-IP.png)

We can access now Jenkins UI from browser, in this example using http://35.237.39.166

![10](images/10-rancher-jenkins-first-access.png)

Under `Build Executor Status` we see two executors being idle, waiting to pick up build jobs. These are coming from Jenkins Master. Since we don't want our master instance to perform build jobs we will disable them. Master instance should be in charge only with scheduling build jobs, distributing the jobs to slaves for actual execution, monitor the slaves and get the build results.
Click on `Manage Jenkins -> Manage Nodes`. Click `Configure` and set `# of executors` to 0. Hit on `Save` and observe that the executors have been removed.

![13](images/13-rancher-jenkins-manage-nodes.png)
![14_1](images/14_1-rancher-jenkins-master-configure.png)
![14_2](images/14_2-rancher-jenkins-master-remove-executors.png)

Click on `Manage Jenkins -> Manage Plugins` and let's check that we have the three plugins installed, the ones from Dockerfile-jenkins-master, most important here is to check the presence of `Kubernetes` plugin. We can see here, that our version, 1.13.0 is installed.

![11](images/11-rancher-jenkins-manage-plugins.png)
![12](images/12-rancher-jenkins-check-plugins.png)

Let's configure now the plugin. For this we need to go in `Manage Jenkins -> Configure System`. Scroll this page to bottom, you will see there the `Cloud` section. Click on `Add a new cloud` and let's start filling the needed information.

![18](images/18-rancher-jenkins-configure-system.png)
![19](images/19-rancher-jenkins-cloud.png)
![20-1](images/20-rancher-jenkins-cloud-1.png)

As you see in the above screenshot, fields that need to be filled are:
- Name
- Kubernetes URL
- Credentials
- Jenkins tunnel

Kubernetes URL and Credentials (which you will need to add them by clicking on the `Add` button) you can get from GCP console. As soon as you added the credentials, test the connection by clicking `Test Connection`, it should return Connection Test Successful. Jenkins tunnel you can take it from `Service Discovery` tab from Rancher.

![16](images/16-rancher-gke-show-credentials.png)
![17](images/17-rancher-gke-get-credentials.png)
![20-3](images/20-rancher-jenkins-cloud-3.png)

Scroll down as we need to add a `Kubernetes Pod Template` and a `Container Template` for the two Jenkins Slave types which we'll use (remember that we have pushed two slave images to docker hub). Click on `Add Pod Template` and fill in the needed information.

Most important thing to follow here:
- labels, they will be different for the two type of Pods: `jenkins-slave-jnlp1` and `jenkins-slave-jnlp2`
- container name, this is mandatory to be set to `jnlp`. (in order to replace the [default JNLP agent](https://hub.docker.com/r/jenkins/jnlp-slave/), the name of the container with the custom JNLP image must be jnlp)
- make sure that you remove the pre-filled info from `Command to run` and `Arguments to pass to the command` and leave the two empty.

![21](images/21-rancher-jenkins-pod-template-1.png)
![22](images/22-rancher-jenkins-pod-template-2.png)

Now we are all set up. Configuration is done so we can create some build jobs to see the power of Jenkins scaling on top of Kubernetes. We will create 5 build jobs for the first image (the one containing `jenkins-slave1` file). In order to enforce build jobs to spin the image we want, we will use Jenkins `Label Expression`. As soon as you fill in the name of the expression you will see a message that the label is serviced by a cloud, the one we configured.

![23](images/23-racher-jenkins-job1.png)
![24](images/24-rancher-jenkins-label-expression.png)
![25](images/25-rancher-jenkins-shell-1.png)

Perform same steps as above, just change the `Label Expression` to match the second image, value in this case should be `jenkins-slave-jnlp2`. Check here as well to use Ansicolor (just adds some color to console output) and fill in the bash commands.

![26](images/26-rancher-jenkins-shell-2.png)

Go to the home screen and start all the ten jobs you just created. As soon as you start them, they will be queued, check them under the `Build Queue` section.

![28](images/28-rancher-jenkins-start-jobs.png)

Takes few seconds and Pods are starting to be created (you can check Rancher's `Workload` tab), therefor executors/slaves are getting attached to the master which instructs them to start processing the queued jobs.

![29](images/29-rancher-jenkins-executors.png)
![30](images/30-rancher-workloads-executors.png)

As soon as executors finish the jobs, they are being removed, so Pods are not available in cluster anymore.

![31](images/31-rancher-remove-pods.png)

To check the status of our build jobs we can randomly pick one from each side. As expected the first jobs have used our first image containing `jenkins-slave1` file, while the other jobs have used the second image containing `jenkins-slave2` file.

![32](images/32-rancher-test-results-1.png)
![33](images/33-rancher-test-results-2.png)

Congratulations! If you’ve reached this point, you've seen the true power of Jenkins auto-scaling inside the Kubernetes cluster!

## Conclusion

Let's recap our exercise:
- we created a clusters using Rancher
- we created custom docker images for Jenkins master and slaves
- we deployed Jenkins master and created a L4 service LoadBalancer
- we configured `kubernetes` plugin so we can have dynamic agents in our Kubernetes cluster.
- we tested a scenarios using a number of build jobs

The main scope behind this article was to highlight the basic configuration for setting up Jenkins Master and Slave architecture. We saw how agents are launched using JNLP and how the image connects automatically to the Jenkins master. To achieve this we used [Rancher](https://rancher.com/) to create the cluster, deploy a workload and then monitor the Pods. Of course the most important piece in this puzzle was [kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin) which glues everything so well together.


