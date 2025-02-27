---
layout: documentation
title: Run flotta on Kind
---

Flotta can run on top of any Kubernetes distributions, for testing and
development you can follow this guide to get started running flotta on your
devices.

Let's start Kind kubernetes cluster

```shell
$ -> cat <<EOF >>config.yaml
---
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 443
    hostPort: 8043
    protocol: TCP
EOF
$ -> kind create cluster --config config.yaml
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind
```

Check minikube status:

```shell
$ -> kubectl cluster-info --context kind-kind

Kubernetes control plane is running at https://127.0.0.1:39435
CoreDNS is running at https://127.0.0.1:39435/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```


Flotta operator has a few tools that helps to install flotta, for that, let's
clone it:

```shell
git clone https://github.com/project-flotta/flotta-operator -b main --depth 1
cd flotta-operator
```

First, we need to install Openshift-router, the ingress that it's supported now:

```shell
$ -> make install-router
kubectl apply -f https://raw.githubusercontent.com/openshift/router/master/deploy/router_rbac.yaml
clusterrole.rbac.authorization.k8s.io/openshift-ingress-router created
clusterrolebinding.rbac.authorization.k8s.io/openshift-ingress-router created
clusterrolebinding.rbac.authorization.k8s.io/openshift-ingress-router-auth-delegator created
namespace/openshift-ingress created
serviceaccount/ingress-router created
kubectl apply -f https://raw.githubusercontent.com/openshift/router/master/deploy/route_crd.yaml
customresourcedefinition.apiextensions.k8s.io/routes.route.openshift.io created
kubectl apply -f https://raw.githubusercontent.com/openshift/router/master/deploy/router.yaml
deployment.apps/ingress-router created
```

Now let's install Flotta on the cluster:

```shell
$ -> make deploy IMG=quay.io/project-flotta/flotta-operator:latest TARGET=k8s
```

A bunch of CRDs are now created, where the definitions can be found
[here](../operations/crd.md):

```
$ ->  kubectl  api-resources | grep flotta
edgeconfigs                                    management.project-flotta.io/v1alpha1   true         EdgeConfig
edgedevices                                    management.project-flotta.io/v1alpha1   true         EdgeDevice
edgedevicesets                                 management.project-flotta.io/v1alpha1   true         EdgeDeviceSet
edgedevicesignedrequest           edsr         management.project-flotta.io/v1alpha1   true         EdgeDeviceSignedRequest
edgeworkloads                                  management.project-flotta.io/v1alpha1   true         EdgeWorkload
playbookexecutions                             management.project-flotta.io/v1alpha1   true         PlaybookExecution
```

At the same time, in the flotta namespace an operator should be running:
```
$ -> kubectl -n flotta get pods
NAME                                                  READY   STATUS    RESTARTS   AGE
flotta-operator-controller-manager-5ffc74fd8c-m7qbp   2/2     Running   0          55s
```

Flotta is now running and ready to register edgedevices. To register a
edgedevice we need first to retrieve the install scripts using the Makefile
target `agent-install-scripts`.

```
$ -> make agent-install-scripts
```

Now, two scripts are created:
  - hack/install-agent-dnf.sh: To install on normal Fedora installations
  - hack/install-agent-rpm-ostree.sh: To install on rpm-ostree devices.

On the device, with Fedora already installed, we need to execute the following:

```
$ sudo hack/install-agent-dnf.sh -i 192.168.2.10
```

Where 192.168.2.10 is your host ip.

After a while, if you list the devices in your cluster, you can see that the
edgedevice is already registered:

```
---> kubectl get edgedevices
NAME        AGE
camera-ny   118s
```

From here, you should be able to deploy workloads to these devices. As explained
[here](./running_workloads.md)
