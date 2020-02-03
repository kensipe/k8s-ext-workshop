# Lab Kubernetes API

## Objective

The focus of this lab is to become familar with the kube-api.  Through this lab you should have an understanding of the kube-api, how it can be accessed and how the API is formed.


1. Start cluster

`kind create cluster`

```bash
kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.17.0) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
⠊⠁ Starting control-plane 🕹️ on
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```

2. Access the API via `kubectl`

Try the following:
`k get --raw /`

Get namespaces
`k get --raw /api/v1/namespaces`
`k get --raw /api/v1/namespaces/default`

```
k get --raw /api/v1/namespaces/default/ 
{"kind":"Namespace","apiVersion":"v1","metadata":{"name":"default","selfLink":"/api/v1/namespaces/default","uid":"288cb0cf-4257-4288-9863-d313bc502972","resourceVersion":"146","creationTimestamp":"2020-02-02T03:15:52Z"},"spec":{"finalizers":["kubernetes"]},"status":{"phase":"Active"}}
```

**tip:** `jq` or `python json.tool` can make this easier to read.
`k get --raw /api/v1/namespaces/default | jq .` or
`k get --raw /api/v1/namespaces/default | python -m json.tool`


3. Time for a proxy

`k proxy`

```bash
 k proxy
Starting to serve on 127.0.0.1:8001
```

`curl localhost:8001`
`curl localhost:8001/api/v1/namespaces/default`

**note:** exit proxy with ctrl+c

4. api-resouces

Lets get a list of resources from `kubectl`
`k api-resources`

Which resources are namespaced? and which are cluster scoped?

For cluster scoped: `k api-resources --namespaced=false`

What is the shortname for `PodSecurityPolicy`?

5. Explaining Resources

Explain is a great way to understand the defined structure of a resource or kind.  This is accomplish through `k explain <kind>`

`k explain ns`

Almost all resources at this high level report roughly the same apiVersion, kind, metadata, spec, status information.  In order to get the full structure of this kind use the `--recursive` flag.

`k explain ns --recursive`

Notice the status field `phase`.  Let's display that as an output.

`k get ns -o custom-columns=NAME:.metadata.name,PHASE:.status.phase`

Example output:
```
NAME                 PHASE
default              Active
kube-node-lease      Active
kube-public          Active
kube-system          Active
local-path-storage   Active
```

Explain is incredibly useful in understanding the structure of types deployed in kubernetes.

## Summary

The Kubernetes API server is the gateway into kubernetes and is accessed via HTTP.   All interactions with Kubernetes is through it.  All controllers / operators work through API for read, update and control. 


todo: use v=9 tracing