# Kubernetes Homelab Configuration

This repository is intended to document all things necessary to configure a
k8s-based homelab exactly as I have running on my local network.

## Node configuration

First up, we'll need to have some nodes with Kubernetes installed. I'll be going
with [k3s](https://k3s.io/), since it seems to be the easiest to set up and run
on a generic Linux box. This guide assumes that you want your Kubernetes resources
remote to a central unix-based developer machine. All commands will be done with
that in mind. The server config will be placed as a header above any commands that
need to be run

### k3s node - Enabling ssh access

If you're anything like me, you don't really want to have to jump between screens
or machines while you're working on this system. So the first thing to do is to
set up SSH access to the node you intend to install k3s on. I'll also include
the commands to set SSH up to run on boot so that you can reboot and not have to
run any commands on the system locally to re-enable SSH

#### From k3s node - enable SSH
```zsh
# Assume root user to make global system changes
sudo su

# Set up SSH daemon to start on boot
systemctl enable sshd.service

# Start SSH daemon for this session, since it won't auto-start
systemctl start sshd.service

# Exit root session
exit
```

#### From developer machine - create key and copy to remote authorized-keys
```zsh
ssh-keygen -t ecdsa -b 521
ssh-copy-id user@k3s-node
```

If you're having trouble finding the private IP of your k3s node, you can run
```zsh
ifconfig
```
and search for something resembling an internal IP

Congrats! You've now set up SSH to your remote server. SSH to the remote node
and let's get to work!

## k3s configuration

Now that we can SSH to our k3s node, there's some basic setup that needs to happen
for our k3s node to function correctly.

### k3s node - Install software

Follow instructions [here](https://docs.k3s.io/installation/requirements?os=rhel) for basic setup
Then, follow the quick-start [guide](https://docs.k3s.io/quick-start) to get everything installed

### Developer machine - set up kubectl

Download kubectl [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management) 
Then, configure kubectl by doing the following:

#### k3s node - Fetch current kubeconfig

```zsh
# Fetch kubeconfig for current cluster
sudo k3s kubectl config view --raw > /tmp/kubeconfig

# Edit kubeconfig - replace 127.0.0.1 with private IP for use from remote systems
nvim /tmp/kubeconfig
```

#### Both machines - SCP file
```zsh
# On developer machine - restart SSH server temporarily to allow SCP
sudo systemctl start sshd.service

# SCP file from k3s node to developer machine
scp /tmp/kubeconfig user@host:/tmp/kubeconfig

# Developer machine - stop SSH listener
sudo systemctl stop sshd.service
```

#### Developer machine - move config to correct location and validate
```zsh
# Move kubeconfig to correct location
mv /tmp/kubeconfig ~/.kube/config

# Check that kubectl is now properly configured
kubectl cluster-info
```

## ArgoCD configuration

For my first CD test here, I'm going to be using ArgoCD as my GitOps tool of choice. 
To install ArgoCD, follow the instructions [here](https://argo-cd.readthedocs.io/en/stable/getting_started/)

**NOTE** - Make sure you install both the k8s components and the CLI

```zsh
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

For initial config/testing, I'm going to port-forward the ArgoCD traffic to my
developer machine

Make sure this command is run in the background, as it will consume a terminal session
```zsh
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial admin password to log in as admin
argocd admin initial-password -n argocd
```

Congrats! You have now deployed a basic ArgoCD setup

## App-of-Apps

The deployment pattern we'd decided to leverage for this cluster is the app-of-apps
pattern, which involves declaring an ArgoCD "Application" resource that describes
manifests for downstream applications, which then get rolled up and managed by Argo
after bootstrapping.

### Bootstrapping the cluster

1. Navigate to the apps directory and copy the "app-of-apps.yaml"

## Networking - deploy Istio

Now that we have our CD framework developed using Argo, we can start deploying
GitOps-enabled applications!
