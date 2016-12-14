# cri-o Tutorial

This tutorial will walk you through the installation of [cri-o](https://github.com/kubernetes-incubator/cri-o), an Open Container Initiative-based implementation of [Kubernetes Container Runtime Interface](https://github.com/kubernetes/kubernetes/blob/master/docs/proposals/container-runtime-interface-v1.md), and the creation of [Redis](https://redis.io/) server running in a [Pod](http://kubernetes.io/docs/user-guide/pods/).

## Prerequisites

A machine is required to download and build the `cri-o` components and run the commands in this tutorial.

Create a machine running Ubuntu 16.10:

```
gcloud compute instances create cri-o \
  --machine-type n1-standard-2 \
  --image-family ubuntu-1610 \
  --image-project ubuntu-os-cloud
```

SSH into the machine:

```
gcloud compute ssh cri-o
```

## Installation

### runc

Download the `runc` release binary:

```
wget https://github.com/opencontainers/runc/releases/download/v1.0.0-rc2/runc-linux-amd64
```

Set the executable bit and copy the `runc` binary into your PATH:

```
chmod +x runc-linux-amd64 
```

```
sudo mv runc-linux-amd64 /usr/bin/runc
```

Print the `runc` version:

```
runc -version
```
```
runc version 1.0.0-rc2
commit: c91b5bea4830a57eac7882d7455d59518cdf70ec
spec: 1.0.0-rc2-dev
```

### cri-o

The `cri-o` project does not ship binary releases so you'll need to build it from source.

#### Install the Go runtime and tool chain

Download the Go 1.7.4 binary release:

```
wget https://storage.googleapis.com/golang/go1.7.4.linux-amd64.tar.gz
```

Install Go 1.7.4:

```
sudo tar -xvf go1.7.4.linux-amd64.tar.gz -C /usr/local/
```

```
mkdir -p $HOME/go/src
```

```
export GOPATH=$HOME/go
```

```
export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin
```

At this point the Go 1.7.4 tool chain should be installed:

```
go version
```

```
go version go1.7.4 linux/amd64
```

#### Build cri-o from source

```
sudo apt-get install -y libglib2.0-dev libseccomp-dev libapparmor-dev
```

```
go get -d github.com/kubernetes-incubator/cri-o
```

```
cd $GOPATH/src/github.com/kubernetes-incubator/cri-o
```

```
make install.tools
```

```
make
```

```
sudo make install
```

Output:

```
install -D -m 755 kpod /usr/bin/kpod
install -D -m 755 ocid /usr/bin/ocid
install -D -m 755 ocic /usr/bin/ocic
install -D -m 755 conmon/conmon /usr/libexec/ocid/conmon
install -D -m 755 pause/pause /usr/libexec/ocid/pause
install -d -m 755 /usr/share/man/man{1,5,8}
install -m 644 docs/kpod.1 docs/kpod-launch.1 -t /usr/share/man/man1
install -m 644 docs/ocid.conf.5 -t /usr/share/man/man5
install -m 644 docs/ocid.8 -t /usr/share/man/man8
install -D -m 644 ocid.conf /etc/ocid/ocid.conf
install -D -m 644 seccomp.json /etc/ocid/seccomp.json
```

### cni

```
go get -d github.com/containernetworking/cni
```

```
cd $GOPATH/src/github.com/containernetworking/cni
```

```
sudo mkdir -p /etc/cni/net.d
```

```
sudo sh -c 'cat >/etc/cni/net.d/10-mynet.conf <<-EOF
{
    "cniVersion": "0.2.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.88.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0"  }
        ]
    }
}
EOF'
```

```
sudo sh -c 'cat >/etc/cni/net.d/99-loopback.conf <<-EOF
{
    "cniVersion": "0.2.0",
    "type": "loopback"
}
EOF'
```

Build the `cni` binaries:

```
./build
```

Output:

```
Building API
Building reference CLI
Building plugins
   flannel
   tuning
   bridge
   ipvlan
   loopback
   macvlan
   ptp
   dhcp
   host-local
   noop
```

Install the `cni` binaries:

```
sudo mkdir -p /opt/cni/bin
```

```
sudo cp bin/* /opt/cni/bin/
```

### docker

Docker is required to pull and store docker images on the local filesystem. The dependency on the docker daemon will go away over time as cri-o will eventually support these features natively.

Download the Docker 

Download the Docker 1.12.4 binary release: 

```
wget https://get.docker.com/builds/Linux/x86_64/docker-1.12.4.tgz
```

Install Docker:

```
tar -xvf docker-1.12.4.tgz
```

```
sudo cp docker/docker* /usr/bin/
```

```
sudo sh -c 'echo "[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
ExecStart=/usr/bin/docker daemon \
  --iptables=false \
  --ip-masq=false \
  --host=unix:///var/run/docker.sock \
  --log-level=error \
  --storage-driver=overlay
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/docker.service'
```

```
sudo systemctl daemon-reload
```

```
sudo systemctl enable docker
```

```
sudo systemctl start docker
```

```
sudo docker version
```

```
Client:
 Version:      1.12.4
 API version:  1.24
 Go version:   go1.6.4
 Git commit:   1564f02
 Built:        Tue Dec 13 02:47:26 2016
 OS/Arch:      linux/amd64

Server:
 Version:      1.12.4
 API version:  1.24
 Go version:   go1.6.4
 Git commit:   1564f02
 Built:        Tue Dec 13 02:47:26 2016
 OS/Arch:      linux/amd64
```

## Pod Tutorial

```
cd $GOPATH/src/github.com/kubernetes-incubator/cri-o
```

```
sudo ocic pod create --config test/testdata/sandbox_config.json
```

```
POD_ID=$(sudo ocic pod create --config test/testdata/sandbox_config.json)
```

> sudo ocic pod create --config test/testdata/sandbox_config.json

```
sudo ocic pod status --id $POD_ID
```

```
ID: cd6c0883663c6f4f99697aaa15af8219e351e03696bd866bc3ac055ef289702a
Name: podsandbox1
UID: redhat-test-ocid
Namespace: redhat.test.ocid
Attempt: 1
Status: SANDBOX_READY
Created: 2016-12-14 15:59:04.373680832 +0000 UTC
Network namespace: /var/run/netns/cni-bc37b858-fb4d-41e6-58b0-9905d0ba23f8
IP Address: 10.88.0.4
Labels:
	group -> test
Annotations:
	owner -> hmeng
	security.alpha.kubernetes.io/seccomp/pod -> unconfined
	security.alpha.kubernetes.io/sysctls -> kernel.shm_rmid_forced=1,net.ipv4.ip_local_port_range=1024 65000
	security.alpha.kubernetes.io/unsafe-sysctls -> kernel.msgmax=8192
```

Run a container inside a pod

```
CONTAINER_ID=$(ocic ctr create --pod $POD_ID --config test/testdata/container_redis.json)
```

> ocic ctr create --pod $POD_ID --config test/testdata/container_redis.json

```
sudo ocic ctr start --id $CONTAINER_ID
```

```
sudo ocic ctr status --id $CONTAINER_ID
```

```
ID: d0147eb67968d81aaddbccc46cf1030211774b5280fad35bce2fdb0a507a2e7a
Name: podsandbox1-redis
Status: CONTAINER_RUNNING
Created: 2016-12-14 16:00:42.889089352 +0000 UTC
Started: 2016-12-14 16:01:56.733704267 +0000 UTC
```

```
sudo ocic ctr stop --id $CONTAINER_ID
```

```
sudo ocic ctr remove --id $CONTAINER_ID
```

```
sudo ocic pod stop --id $POD_ID
```

```
sudo ocic pod remove --id $POD_ID
```

```
sudo ocic pod list
```

```
sudo ocic ctr list
```
