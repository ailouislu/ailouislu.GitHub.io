---
layout: post

title: "Istio Sidecar Auto Injection Mechanism"
subtitle: "Analysis of Kubernetes Webhook Extension Mechanism"
description: "Kubernetes 1.9 introduced the Admission Webhook extension mechanism. Through Webhook, developers can flexibly extend the functionality of the Kubernetes API Server, validating or modifying resources when creating them. Istio 0.7 leveraged Kubernetes webhook to implement automatic sidecar injection."
excerpt: "Kubernetes 1.9 introduced the Admission Webhook extension mechanism. Through Webhook, developers can flexibly extend the functionality of the Kubernetes API Server, validating or modifying resources when creating them. Istio 0.7 leveraged Kubernetes webhook to implement automatic sidecar injection."
date: 2023-01-05
author: "Louis Lu"
image: "/img/post-bg-coffee.jpeg"
published: true
tags:
  - Kubernetes
  - Istio
URL: "/2018/05/23/istio-auto-injection-with-webhook/"
categories: [Tech]
---

## Introduction

Kubernetes 1.9 introduced the Admission Webhook extension mechanism, allowing developers to extend the functionality of the Kubernetes API Server flexibly. This extension occurs during the creation of resources, where the resources can be validated or modified.

The advantage of using a webhook is that it doesn't require modification and recompilation of the API Server's source code to extend its functionality. The inserted logic is implemented as an independent web process, passed as parameters to Kubernetes, and called back by Kubernetes during its own logic processing.

Istio 0.7 version utilized the Kubernetes webhook to achieve automatic sidecar injection.

<!--more-->

## What is Admission

Admission is a term in Kubernetes, referring to a phase in the resource request process for the Kubernetes API Server. As shown in the diagram below, when the API Server receives a resource creation request, it undergoes authentication, authorization, Admission processing, and finally, the resource is saved to etcd.
![](/img/2018-4-25-istio-auto-injection-with-webhook/admission-phase.png)

In Admission, there are two crucial phases, Mutation and Validation, with the following logic executed in these phases:

- Mutation

  Mutation, as the name suggests, allows modifications to the request content.

- Validation

  In the Validation phase, modifying the request content is not allowed, but based on the content, the decision to proceed or reject the request can be made.

## Admission Webhook

---

Through the Admission webhook, you can add both Mutation and Validation types of webhook plugins. These plugins, along with Kubernetes' precompiled Admission plugins, share the same capabilities. Possible use cases include:

- Modifying resources, e.g., Istio uses the Admin Webhook to add Envoy sidecar containers to Pod resources.
- Custom validation logic, e.g., imposing special requirements on resource names or validating the legitimacy of custom resources.

## Automatically Injecting Istio Sidecar Using Webhook

---

### Kubernetes Version Requirements

Webhook support requires Kubernetes 1.9 or higher. Confirm that the kube-apiserver's Admission webhook feature is enabled using the following command:

```
kubectl api-versions | grep admissionregistration

admissionregistration.k8s.io/v1beta1
```

### Generate Key and Certificate for Sidecar Injection Webhook

Webhook uses digital certificates for authentication with kube-apiserver. Therefore, you need to generate a key pair using a tool and apply for a digital certificate from Istio CA.

```
./install/kubernetes/webhook-create-signed-cert.sh /
    --service istio-sidecar-injector /
    --namespace istio-system /
    --secret sidecar-injector-certs
```

### Configure the Generated Digital Certificate in the Webhook

```
cat install/kubernetes/istio-sidecar-injector.yaml | /
     ./install/kubernetes/webhook-patch-ca-bundle.sh > /
     install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

### Create Sidecar Injection ConfigMap

```
kubectl apply -f install/kubernetes/istio-sidecar-injector-configmap-release.yaml
```

### Deploy Sidecar Injection Webhook

```
kubectl apply -f install/kubernetes/istio-sidecar-injector-with-ca-bundle.yaml
```

Check the deployed webhook injector using the following command:

````
kubectl -n istio-system get deployment -listio=sidecar-injector
Copy
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-sidecar-injector   1         1         1            1           1d
```

### Enable Automatic Sidecar Injection for the Namespace

```
kubectl label namespace default istio-injection=enabled

kubectl get namespace -L istio-injection

NAME           STATUS    AGE       ISTIO-INJECTION
default        Active    1h        enabled
istio-system   Active    1h
kube-public    Active    1h
kube-system    Active    1h
```

```

###References
- [Extensible Admission is Beta](https://kubernetes.io/blog/2018/01/extensible-admission-is-beta)
- [Installing the Istio Sidecar](https://istio.io/docs/setup/kubernetes/sidecar-injection.html)
````
