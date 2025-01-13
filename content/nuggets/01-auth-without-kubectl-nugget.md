---
title: "Nugget: Check Kubernetes Pod Logs Without kubectl"
description: "Authenticating to K8s cluster without kubectl"
image: "https://github.com/TaskMasterErnest/smol/tree/master/images/tskmstr.jpg"
date: 2025-01-11T12:00:00+00:00
draft: false
tags: ["kubernetes", "debugging", "tips"]
categories: ["devops"]
---

Ever needed to check pod logs but couldn't use `kubectl`? Here's a quick way using the Kubernetes API directly:

```bash
curl -k \
  -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://kubernetes.default.svc/api/v1/namespaces/default/pods/<pod-name>/log
```

Breaking down this command:
1. The name `kubernetes.default.svc` is the DNS name that points to the API server in the Kubernetes cluster. It is a part of the built-in service discovery system in the cluster.
2. The path `/api/v1/namespaces/default/pods/<pod-name>/log` follows the RESTful nature of the way the API server is exposed:
- the `/api/v1` specifies the API version.
- the `/namespaces/<namespace>` indicates the namespace we are looking in.
- the `/pods/<pod-name>/log` requests the logs for the specific Pod.

This method is useful in scenarios where:
- the `kubectl` binary is not available in the environment.
- you have automation scripts that need to access logs.
- a case where you are troubleshooting kubectl itself.

This essentially expands to how you can access a Kubernetes cluster without having the `kubectl` tool.