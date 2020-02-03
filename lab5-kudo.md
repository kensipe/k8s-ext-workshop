# Lab KUDO

## Objective

The focus of this lab is to become familar with KUDO.  Through this lab you hand craft an operator, create an operator repository and install the operator to your kubernetes cluster.  

## Prerequisites

* Running Kubernetes 1.15+ cluster

---

## Create an Operator from Scratch

This is a step-by-step walk through of the creation of an operator using the KUDO CLI to generate the KUDO operator structure.


### Create the Core Operator Structure

```bash
# create operator folder
mkdir first-operator
cd first-operator
kubectl kudo package new first-operator
```

This creates the main structure of the operator which can be viewed using the `tree` command:

```bash
$ tree .
.
└── operator
    ├── operator.yaml
    └── params.yaml
```

::: tip Note
Use the `-i` flag with `kubectl kudo package new` to be prompted interactively for operator details.
:::

### Add a Maintainer

`kubectl kudo package add maintainer "your name" your@email.com`

### Add a Task

`kubectl kudo package add task`

This command uses an interactive prompt to construct the details of the task.  Here is an example interaction:

```bash
$ kubectl kudo package add task
Task Name: app
✔ Apply
Task Resource: deployment
✗ Add another Resource:
```

### Add a Plan

`kubectl kudo package add plan`

This command uses an interactive prompt to construct the details of the plan.  Here is an example interaction:

```bash
$ kubectl kudo package add plan
✔ Plan Name: deploy
✔ serial
Phase 1 name: main
✔ parallel
Step 1 name: everything
✔ app
✗ Add another Task:
✗ Add another Step:
✗ Add another Phase:
```

### Add a Parameter

`kubectl kudo package add parameter`

This command uses an interactive prompt to construct the details of the parameter.  Here is an example interaction:

```bash
$ kubectl kudo package add parameter
Parameter Name: replicas
Default Value: 2
Display Name:
Description: Number of replicas that should be run as part of the deployment
✔ false
✗ Add Trigger Plan:
```

These steps have created the entirety of the first-operator with the exception of the details in the `template/deployment.yaml` file.  To complete this operator execute the following:

```bash
cat << EOF > operator/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: {{ .Params.replicas }}
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.7.9
          ports:
            - containerPort: 80
EOF
```

---

## Package a KUDO operator

In order to distribute a KUDO operator the files are packaged together in a compressed tarball.  The KUDO CLI provides a mechanism to create this package format while verifying the integrity of the operator.

### Package KUDO Operator

```bash
rm -rf ~/repo
mkdir -p ~/repo
kubectl kudo package create repository/first-operator/operator/ --destination=~/repo
```

::: warning Potential Data Loss
You may want to check the contents of the `~/repo` folder prior to deleting it.
:::

The output looks like:

```bash
kubectl kudo package create repository/first-operator/operator/ --destination=~/repo
package is valid
Package created: /Users/kensipe/repo/first-operator-0.2.0.tgz
```

### Check to see the operator is built

```bash
ls ~/repo
first-operator-0.2.0.tgz
```

---

## Initialize KUDO in a Cluster

The objective of this lab is to initialize KUDO in a Kubernetes cluster.

### Initialize KUDO

`kubectl kudo init --wait`

This results in:

1. the deployment of KUDO CRDs
2. the creation of kudo-system namespace
3. deployment of the kudo controller

Output of a KUDO init will look like the following:

```bash
$ kubectl kudo init
✅ installed crds
✅ installed service accounts and other requirements for controller to run
✅ installed kudo controller
```

### Check to see KUDO Manager is running

The installation of KUDO is verified by confirming that the `kudo-controller-manager-0` is in a running status.

```bash
$ kubectl get -n kudo-system pod
NAME                        READY   STATUS    RESTARTS   AGE
kudo-controller-manager-0   1/1     Running   0          11m
```

---

## Host an Operator in a local repository

This lab explains how to host an operator repository on your local system.

### Build Local Index File

`kubectl kudo repo index ~/repo`

### Run Repository HTTP Server

```bash
cd ~/repo
python -m http.server 80
```

### Add the local repository to KUDO client

`kubectl kudo repo add local http://localhost`

### Set the local repository to default KUDO context

`kubectl kudo repo context local`

### Confirm KUDO context

```bash
$ kubectl kudo repo list
NAME     	URL
community	https://kudo-repository.storage.googleapis.com/0.10.0
*local   	http://localhost
```

::: tip Note
The `*` next to local indicates that it is the default context for the KUDO client.
:::

### Verify you are using the local repository for an installation

Using the verbose CLI output flag (`-v`) with KUDO it is possible to trace from where an operator is being installed from.

`kubectl kudo install first-operator -v 9`

The output should look like:

```bash
$ kubectl kudo install first-operator -v 9
repo configs: { name:community, url:https://kudo-repository.storage.googleapis.com/0.10.0 },{ name:local, url:http://localhost }

repository used { name:local, url:http://localhost }
configuration from "/Users/kensipe/.kube/config" finds host https://127.0.0.1:32768
acquiring kudo client
getting package crds
no local operator discovered, looking for http
no http discovered, looking for repository
getting package reader for first-operator, _
repository using: { name:local, url:http://localhost }
attempt to retrieve package from url: http://localhost/first-operator-0.2.0.tgz
first-operator is a repository package from { name:local, url:http://localhost }
operator name: first-operator
operator version: 0.2.0
parameters in use: map[]
operator.kudo.dev/first-operator unchanged
instance first-operator-instance created in namespace default
instance.kudo.dev/v1beta1/first-operator-instance created
```

You will also see in the terminal running python http.server the following:

```bash
127.0.0.1 - - [14/Jan/2020 07:59:24] "GET /index.yaml HTTP/1.1" 200 -
127.0.0.1 - - [14/Jan/2020 07:59:24] "GET /first-operator-0.2.0.tgz HTTP/1.1" 200 -
```
