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
Then, follow the quick-start [guide](https://docs.k3s.io/quick-start) to get everything installed.
The k3s command that finally worked for me was:
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

### Developer machine - set up kubectl

Download kubectl [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-using-native-package-management) 

### k3s node - Fetch current kubeconfig

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

## General Philosophy - Adding apps to a cluster, building automation for apps

For tinkering purposes, it's often easier to follow a tutorial with cleanup, then
build automation. That is the approach we'll take here.

## ArgoCD configuration

ArgoCD is the CD tool I've chosen to use for this cluster. There are many
others you could pick, but all instructions from here on out will assume ArgoCD
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

Congrats! You have now deployed a basic ArgoCD setup that can be accessed on port 8080

## App-of-Apps & bootstrapping

The deployment pattern we'd decided to leverage for this cluster is the app-of-apps
pattern, which involves declaring an ArgoCD "Application" resource that describes
manifests for downstream applications, which then get rolled up and managed by Argo
after bootstrapping.

### CRD installation

**NOTE** Make sure any CRDs needed by your applications are pre-installed
For our cluster, we'll need the Gateway CRD pre-installed to take advantage of the
Kubernetes Gateway API

Gateway (Istio)
```zsh
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml; }
```

### Repository configuration

We'll also be using the [multiple sources for an application](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
pattern with Argo to enable us to use public Helm charts combined with private values files
stored in a private repository.

To enable Argo to authenticate to our private repository, we simply have to
1. Go to Settings > Repository Configuration
2. Fill in the appropriate settings for this repository

### Application Health Monitoring

I'm using Argo to bootstrap my entire cluster. As a result, I need to make sure that
my cluster's core services (networking, mainly) come up before the rest of the cluster.
To do this, I need to monitor the health of child applications and use SyncWaves
to enforce a certain deployment order. To configure this, run the following line
from within the `argo-app-health-check-fix` directory
```zsh
kubectl patch cm/argocd-cm --type=merge -n argocd --patch-file argocd-cm-patch.yml
```

### Cluster deployment

Now that we've got our CRDs created and our repository credentials in ArgoCD,
it's time to deploy our cluster!

1. In Argo, click the "new app" button and then click "edit as YAML"
2. In the repository, copy the "app-of-apps.yaml" manifest
3. Paste the manifest into Argo, click save, and click create
4. Deploy! Congrats! You've successfully deployed a copy of my homelab cluster

The rest of this README will provide an overview of some of the applications deployed
by our app-of-apps, what they do, and basic concepts/configuration I thought
would be useful fo reference later on.

## Istio - A k8s-native service mesh for secure, centralized networking

This section describes core concepts of the Istio ambient mode service mesh that
I used to develop the working manifests to deploy Istio via ArgoCD

### Istio Ambient mode components

#### Istio-base

The base app. Not much to see here

#### Istiod

The control plane daemonset for Istio

#### Istio CNI

The CNI plugin for Istio. This is an important one, as this will create container
interfaces for any pods launched in istio-enabled namespaces. There are several
issues that can happen with this

#### Istio Ztunnel

Rust-based zero-trust tunneling daemonset for transmitting traffic through the mesh

#### Kubernetes Gatetway API

A custom set of resources for defining traffic routes, provided by the k8s project

#### How traffic gets from the frontend to the backend

If you ever want to visualize this for yourself, [Kiali](https://kiali.io/) is a 
fantastic tool for doing so!

#### Adding applications to the mesh

To add applications to the service mesh, you can either label the namespace or the
application itself using the `istio.io/dataplane-mode=ambient` label

#### Controlling access between applications - SPIFFE IDs

Istio ambient uses the concept of [SPIFFE](https://spiffe.io/docs/latest/spiffe-about/spiffe-concepts/) IDs
to manage workload identities.

Now that we have our CD framework developed using Argo, we can start deploying
GitOps-enabled applications!

Our first application is [Istio](https://istio.io/latest/docs/ambient/install/helm/#install-the-control-plane) -
an ambient service mesh that provides networking APIs for Kubernetes resources.
Charts and k3s-compatible values for Istio are defined in the istio-base, istio-cni, istiod, and istio-ztunnel
application manifests


### A note about Istio and sync waves

For Istio to work correctly, we have to install istio-base, then istiod, then ztunnel. We do this using [sync waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)

There is a bit of config we have to do in ArgoCD to enable it to work with Istio.
Since I haven't figured out a good way to manage Argo with itself, this part was applied manually,
following the instructions in the Argo [documentation](https://argo-cd.readthedocs.io/en/latest/operator-manual/ingress/#istio).

The kustomization.yml and patch.yml that worked when applied to my cluster are in the manual-argo-kustomize directory

### Generating a self-signed cert for local testing

If you're anything like me, you don't super want to expose your cluster to the
Internet until you're reasonably sure you've gotten everything secure. This means
you're doing a lot of either port forwarding or generating self-signed certificates.
Here's some sample commands that should help with generating those certs for
your node computer.

#### Creating self-signed cert w/ openssl

Create the cert on developer machine
```zsh
openssl req -x509 -out node.crt -keyout localhost.key \
  -newkey rsa:2048 -nodes -sha256 \
  -subj '/CN=cluster.local' -extensions EXT -config <( \
   printf "[dn]\nCN=cluster.local\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:cluster.local\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
```

Create Kubernetes secret from cert
```
kubectl create secret tls -n argocd argocd-server-tls --cert=/path/to/node.crt --key=/path/to/localhost.key
```

