# Installing kubernetes in a docker container

zreigz is awesome! Scroll down [this](https://github.com/kubernetes/kubernetes/issues/35712) post to see the detail instruction on how to create the image. And scroll further to see how we can deploy a sample application.

The only change I have made is I have added the commands to be executed in the container in a shell script and copied the same from the docker file. So, after you start and login the container you need to run the script as:

```
# /tmp/kubeadm_install.sh
```

Rest are exactly the same as mentioned in the post.

Just in case the link goes downi I copied the content from the post above:

> zreigz said:

There are two main issues regarding to installation kubeadm in Docker container. First is systemd running in container. Second is installation docker inside container. Successfully the problems were fixed. Here is the Dockerfile which must be used to prepare Ubuntu image

```
FROM ubuntu
ENV container docker
RUN apt-get -y update

RUN apt-get update -qq && apt-get install -qqy \
    apt-transport-https \
    ca-certificates \
    curl \
    lxc \
    vim \
    iptables

RUN curl -sSL https://get.docker.com/ | sh

RUN (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -f $i; done); \
rm -f /lib/systemd/system/multi-user.target.wants/*;\
rm -f /etc/systemd/system/*.wants/*;\
rm -f /lib/systemd/system/local-fs.target.wants/*; \
rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
rm -f /lib/systemd/system/basic.target.wants/*;\
rm -f /lib/systemd/system/anaconda.target.wants/*;

VOLUME /sys/fs/cgroup
VOLUME /var/run/docker.sock
CMD /sbin/init
```

I use this command to build the image in the directory containing the Dockerfile

```
docker build -t kubeadm_docker .
```

Now you can run prepared image and finish kubeadm installation.
Use the following command to run kubeadm_docker image:

```
docker run -it -e "container=docker" --privileged=true -d --security-opt seccomp:unconfined --cap-add=SYS_ADMIN -v /sys/fs/cgroup:/sys/fs/cgroup:ro -v /var/run/docker.sock:/var/run/docker.sock  kubeadm_docker /sbin/init
```

Find running container ID

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
7dd73057620d        kubeadm_docker      "/sbin/init"        About an hour ago   Up About an hour                        furious_fermi
```

Now you can open container console:

```
docker exec -it 7dd73057620d /bin/bash
```

This is your script (with small modifications) to install kubeadm

```
echo "Updating Ubuntu..."
apt-get update -y
apt-get upgrade -y

systemctl start docker

echo "Install os requirements"
apt-get install -y \
  curl \
  apt-transport-https \
  dialog \
  python \
  daemon

echo "Add Kubernetes repo..."
sh -c 'curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -'
sh -c 'echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list'
apt-get update -y

echo "Installing Kubernetes requirements..."
apt-get install -y \
  kubelet

# This is temporary fix until new version will be released
sed -i 38,40d /var/lib/dpkg/info/kubelet.postinst

apt-get install -y \
  kubernetes-cni \
  kubectl \
  kubeadm
```

And finally you can execute

```
# kubeadm init
```

Everything works the same like on local machine.
Good luck :)

## Deploying a sample application

First, create a custom namespace where the application willl ve deployed.
```
$ kubectl create namespace my-app
```
Then, create the deployment yaml file which will be used for creating the application:

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nodetest-01
  labels:
    name: nodetest
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: nodetest
    spec:
      containers:
      - name: nodetest-01
        image: onlydevelop/node-test:0.1
        ports:
        - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: nodetest-01
  labels:
    name: nodetest
spec:
  type: LoadBalancer
  ports:
    # the port that this service should serve on
  - port: 80
    targetPort: 3000
  selector:
    name: nodetest
```

Please note the first section is for the Deployment and the second section is for the Service, which means exposing the application to outside. Few points which could be of interest:

- `replicas` are number of instances running
- `type LoadBalancer` tag creates a load balancer and automatically assigns a domain name for that.
- `image` is the Docker image for the application - I used a sample image created [here](https://github.com/onlydevelop/docker-node)
- `port` is which will be accessed from external and `targetPort` is the container port where the service is running.  

Then, deploy the application using the following command:

```
$ kubectl apply -n  my-app -f "https://github.com/onlydevelop/kubeadm-docker/blob/master/node-test-01.yaml?raw=true"
```

Now, you can login to the dashboard and select the `namespace` as `my-app` and you can go to the `Services` link in the left panel and after a while you will see the external endpoint is available. Please give some time for the DNS to be propagated and then you will be able to see the app like this:

```
$ curl -i http://<your-endpoint>/user/dipanjan
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 48
ETag: W/"30-3lzPh/kc2mUtq4TE59kDaQ"
Date: Fri, 18 Nov 2016 11:48:40 GMT
Connection: Keep-Alive
Age: 0

Hello dipanjan from nodetest-01-2756364007-5u5q8
```

which is the example App we have created [here](https://github.com/onlydevelop/docker-node)

And finally, delete the application by the following command:

```
$ kubectl delete namespace my-app # create a custom namespace
```

## Creating a Replication controller

We will create a replication controller first to with a sample size of 2

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nodetest-01
  labels:
    name: nodetest
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: nodetest
    spec:
      containers:
      - name: nodetest-01
        image: onlydevelop/node-test:0.1
        ports:
        - containerPort: 3000
```

And then, we will create the controller:

```
$ kubectl create -n my-app -f node-test-01-rc.yaml  
$ kubectl get -n my-app rc

NAME          DESIRED   CURRENT   READY     AGE
nodetest-01   2         2         2         2m
```

You can check from the dashboard that the replication controller is created. 

Then, scale the controller with replicas=3 and check if it increased:

```
$ kubectl scale rc -n my-app nodetest-01 --replicas=3
$ kubectl get -n my-app rc

NAME          DESIRED   CURRENT   READY     AGE
nodetest-01   3         3         3         6m
```


