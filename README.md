# Installing kubernetes in a docker container

zreigz is awesome! Scroll down [this](https://github.com/kubernetes/kubernetes/issues/35712) post to see the detail instruction on how to create the image.

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

