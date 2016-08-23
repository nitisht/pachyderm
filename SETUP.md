# Setup

## Intro

Pachyderm is built on [Kubernetes](http://kubernetes.io/).  As such, technically Pachyderm can run on any platform that Kubernetes supports.  This guide covers the following commonly used platforms:

* [Local](#local-deployment)
* [Google Cloud Platform](#google-cloud-platform)
* [AWS](#amazon-web-services-aws)
* [OpenShift](#openshift)

Each section starts with deploying Kubernetes on the said platform, and then moves on to deploying Pachyderm on Kubernetes.  If you have already set up Kubernetes on your platform, you may directly skip to the second part.

## Common Prerequisites

- [FUSE (optional)](#fuse-optional) >= 2.8.2
- [Kubectl (kubernetes CLI)](#kubectl) >= 1.2.2
- [pachctl](#pachctl)
- [Pachyderm Repository](#pachyderm)

### FUSE (optional)

Having FUSE installed allows you to mount PFS locally, which can be nice if you want to play around with PFS.

FUSE comes pre-installed on most Linux distributions.  For OS X, install [OS X FUSE](https://osxfuse.github.io/)

### Kubectl

Make sure you have version 1.2.2 or higher.

```shell
### Darwin (OS X)
$ wget https://storage.googleapis.com/kubernetes-release/release/v1.2.2/bin/darwin/amd64/kubectl

### Linux
$ wget https://storage.googleapis.com/kubernetes-release/release/v1.2.2/bin/linux/amd64/kubectl

### Copy kubectl to your path
chmod +x kubectl
mv kubectl /usr/local/bin/
```

### pachctl

`pachctl` is a command-line utility used for interacting with a Pachyderm cluster.

#### Installation

##### Homebrew

```shell
$ brew tap pachyderm/tap && brew install pachctl
```

##### Deb Package

If you're on linux 64 bit amd, you can use our pre-built deb package like so:

```shell
$ curl -o /tmp/pachctl.deb -L https://pachyderm.io/pachctl.deb && dpkg -i /tmp/pachctl.deb
```

##### From Source

You'll need Go 1.6, which you can find [here](https://golang.org/doc/install).

To install pachctl from source, we assume you'll be compiling from within $GOPATH. So to install pachctl do:

```shell
$ go get github.com/pachyderm/pachyderm
$ cd $GOPATH/src/github.com/pachyderm/pachyderm
$ make install
```

Make sure you add `GOPATH/bin` to your `PATH` env variable:

```shell
$ export PATH=$PATH:$GOPATH/bin
```

### Pachyderm

Even if you haven't installed `pachctl` from source, you'll need some make tasks located in the pachyderm repositoriy. If you haven't already cloned the repo, do so:

```shell
$ git clone  git@github.com:pachyderm/pachyderm
```

## Local Deployment

### Prerequisites

- [Docker](https://docs.docker.com/engine/installation) >= 1.10

### Port Forwarding

Both kubectl and pachctl need a port forwarded so they can talk with their servers.  If your Docker daemon is running locally you can skip this step.  Otherwise (e.g. you are running Docker through [Docker Machine](https://docs.docker.com/machine/)), do the following:


```shell
$ ssh <HOST> -fTNL 8080:localhost:8080 -L 30650:localhost:30650
```

### Deploy Kubernetes

From the root of this repo you can deploy Kubernetes with:

```shell
$ make launch-kube
```

This step can take a while the first time you run it, since some Docker images need to be pulled.

### Deploy Pachyderm

From the root of this repo you can deploy Pachyderm on Kubernetes with:

```shell
$ make launch
```

This step can take a while the first time you run it, since a lot of Docker images need to be pulled.

## Google Cloud Platform

Google Cloud Platform has excellent support for Kubernetes through the [Google Container Engine](https://cloud.google.com/container-engine/).

### Prerequisites

- [Google Cloud SDK](https://cloud.google.com/sdk/) >= 106.0.0

If this is the first time you use the SDK, make sure to follow through the [quick start guide](https://cloud.google.com/sdk/docs/quickstarts).

After the SDK is installed, run:

```shell
$ gcloud components install kubectl
```

### Set up the infrastructure

Pachyderm needs a [container cluster](https://cloud.google.com/container-engine/), a [GCS bucket](https://cloud.google.com/storage/docs/), and a [persistent disk](https://cloud.google.com/compute/docs/disks/) to function correctly.  We've made this very easy for you by creating the `make google-cluster` helper, which will create all of these resources for you.

First of all, set the required environment variables. Choose a name for both the bucket and disk, as well as a capacity for the disk (in GB):

```shell
$ export BUCKET_NAME=some-unique-bucket-name
$ export STORAGE_NAME=pach-disk
$ export STORAGE_SIZE=200
```

You may need to visit the [Console] to fully initialize Container Engine in a new project. Then, simply run the following command:

```shell
$ make google-cluster
```

This creates a Kubernetes cluster named "pachyderm", a bucket, and a persistent disk.  To check that everything has been set up correctly, try:

```shell
$ gcloud compute instances list
# should see a number of instances

$ gsutil ls
# should see a bucket

$ gcloud compute disks list
# should see a number of disks, including the one you specified
```

### Deploy Pachyderm

First of all, record the external IP address of one of the nodes in your Kubernetes cluster:

```shell
$ gcloud compute instances list
```

Then export it with port 30650:

```shell
$ export ADDRESS=[the external address]:30650
# for example:
# export ADDRESS=104.197.179.185:30650
```

This is so we can use [`pachctl`](#pachctl) to talk to our cluster later.

Now you can deploy Pachyderm with:

```shell
$ make google-cluster-manifest > manifest
$ make MANIFEST=manifest launch
```

It may take a while to complete for the first time, as a lot of Docker images need to be pulled.

## Amazon Web Services (AWS)

### Prerequisites

- [AWS CLI](https://aws.amazon.com/cli/)

### Deploy Kubernetes

Deploying Kubernetes on AWS is still a relatively lengthy and manual process comparing to doing it on GCE.  However, here are a few good tutorials that walk through the process:

* http://kubernetes.io/docs/getting-started-guides/aws/
* https://coreos.com/kubernetes/docs/latest/kubernetes-on-aws.html

### Set up the infrastructure

First of all, set these environment variables:

```shell
$ export KUBECTLFLAGS="-s [the IP address of the node where Kubernetes runs]"
$ export BUCKET_NAME=[the name of the bucket where your data will be stored; this name needs to be unique across the entire AWS region]
$ export STORAGE_SIZE=[the size of the EBS volume that you are going to create, in GBs]
$ export AWS_REGION=[the AWS region where you want the bucket and EBS volume to reside]
$ export AWS_AVAILABILITY_ZONE=[the AWS availability zone where you want your EBS volume to reside]
```

Then, simply run:

```shell
$ make amazon-cluster
```

Record the "volume-id" in the output, then export it:

```shell
$ export STORAGE_NAME=[volume id]
```

Now you should be able to see the bucket and the EBS volume that are just created:

```shell
aws s3api list-buckets --query 'Buckets[].Name'
aws ec2 describe-volumes --query 'Volumes[].VolumeId'
```

### Deploy Pachyderm

First of all, get a set of [temporary AWS credentials](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html):

```shell
$ aws sts get-session-token
```

Then run the following commands with the credentials you get:

```shell
$ AWS_ID=[access key ID] AWS_KEY=[secret access key] AWS_TOKEN=[session token] make amazon-cluster-manifest > manifest
$ make MANIFEST=manifest launch
```

It may take a while to complete for the first time, as a lot of Docker images need to be pulled.

## OpenShift

[OpenShift](https://www.openshift.com/) is a popular enterprise Kubernetes distribution.  Pachyderm can run on OpenShift with two additional steps:

1. Make sure that priviledge containers are allowed (they are not allowed by default):  `oc edit scc` and set `allowPrivilegedContainer: true` everywhere.
2. Remove `hostPath` everywhere from your cluster manifest (e.g. `etc/kube/pachyderm-versioned.json` if you are deploying locally).

Problems related to OpenShift deployment are tracked in this issue: https://github.com/pachyderm/pachyderm/issues/336

## pachctl

### Usage

If Pachyderm is running locally, you are good to go.  Otherwise, you need to make sure that `pachctl` can find the node on which you deployed Pachyderm:

```shell
$ export ADDRESS=[the IP address of the node where Pachyderm runs]:30650
# for example:
# export ADDRESS=104.197.179.185:30650
```

Now, create an empty repo to make sure that everything has been set up correctly:

```shell
pachctl create-repo test
pachctl list-repo
# should see "test"
```

## Next Step

Ready to jump into data analytics with Pachyderm?  Head to our [quick start guide](examples/fruit_stand/README.md).

## Trouble Shooting

### On first deployment of pachd, see CrashLoopBackoff errors

This is usually normal. Until the rethink service comes up and the `pachd` pods can connect, they will crash and backoff. It usually takes about a minute for the cluster to come up. The first time may be longer since docker will need to download some new images.

### Using a custom manifest you see out of date features

It's likely that you need to update the `imagePullPolicy` field(s) for `pachyderm/pachd`. The new default is `Always` so if you're trying to use newly compiled versions, it will pull the version released on Docker Hub (almost certainly older than what is in the repo), so you should set it to `IfNotPresent`

### pachd or pachd-init crash loop with "error connecting to etcd"

This error normally occurs due to Kubernetes services not function because the
kernel does not support iptables. Generally you can solve this with:

```
modprobe netfilter_xt_match_statistic netfilter_xt_match_recent
```

However in other cases it may require recompiling the kernel.  Please head to
[this issue](https://github.com/pachyderm/pachyderm/issues/458) if you're
having trouble with this so we can collect solutions to the problem in one
place.

We'll update this section of the guid as we learn more about this issue.