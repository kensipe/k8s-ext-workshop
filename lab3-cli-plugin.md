# Lab Kubernetes kubectl CLI plugin

## Objective

The focus of this lab is to become familar with kubectl plugin development.  Through this lab you great a plugin that will interact with a Kubernetes cluster. 

## Prerequisites

* Running Kubernetes 1.15+ cluster
* kubectl installed

---

Step 1: clone https://github.com/codementor/k8s-cli

`git clone https://github.com/codementor/k8s-cli`

Or fork and clone 

Step 2: Compile

Make sure the environment is setup correct by `make cli` which will build the cli or `make` which will build and test.  

Step 3. Run CLI from Go

`go run cmd/kubectl-example/main.go`

or

`go run cmd/kubectl-example/main.go version`
will run the project.

Step 3: Build and Install

`make cli-install` will build and install the new example plugin into `$GOPATH/bin`.  Assuming that is in your `$PATH`, the following should now be possible.

```bash
k example version
Example Version: version.Info{GitVersion:"0.1.0", GitCommit:"64e04046", BuildDate:"2020-01-30T22:00:17Z", GoVersion:"go1.13.7", Compiler:"gc", Platform:"darwin/amd64"}
```

Step 4: Test against your cluster

Running `k example resources` while kubectl context is set to an active cluster should result in the following:

```bash
k example resources
Name                           	Namespaced	Kind                          
replicationcontrollers         	true      	ReplicationController         
namespaces                     	false     	Namespace                     
resourcequotas                 	true      	ResourceQuota                 
configmaps                     	true      	ConfigMap                     
pods                           	true      	Pod                           
nodes                          	false     	Node                          
services                       	true      	Service                       
persistentvolumeclaims         	true      	PersistentVolumeClaim         
secrets                        	true      	Secret                        
serviceaccounts                	true      	ServiceAccount                
persistentvolumes              	false     	PersistentVolume   
...
```

Step 5: Explore the Project

Start with `root.go` and understand how commands in Go are created.
Look at `main.go` which is the starting point but for this project isn't very interesting.


Step 6: Add pod listing code which uses structure object references in go.

At line 71 of `pod_list.go` replace `fmt.Printf("add pod list code using direct object references\n")` with the following:

First acquire a kube client and a pods client:

```go
	client := env.NewClientSet(&Settings)
	podsClient := client.CoreV1().Pods(apiv1.NamespaceDefault)
```

With the `podClient` we can query for a list

```go
	list, err := podsClient.List(metav1.ListOptions{})
	if err != nil {
		return err
	}
```

The list that is return is an object which has type information AND contains a list of the pod objects we are interested in.  This is a common pattern in kubernetes object access.  The follow code checks to see if there are any objects and prints a line for each pod found.

```go
	if len(list.Items) == 0 {
		fmt.Printf("no pods discovered\n")
		return nil
	}
	for _, item := range list.Items {
			fmt.Fprintf(p.out, "pod %v in namespace: %v\n", item.Name, item.Namespace)		
	}
```

Step 7. Command Flags and Pod Status

Lets add a new command flag for this list command.  

Find in the `pod_list.go` file where it reads `// status boolean` and add the following code `status bool` to the struct.

Find in the `pod_list.go` file where it reads `// status flag` and add the following code to add a flag 

```go
	f := cmd.Flags()
	f.BoolVarP(&pkg.status, "status", "i", true, "display status info")

```

Now lets change the previous steps code such that a flag will provide a different output.

```go
		if p.status {
			fmt.Fprintf(p.out, "pod %v in namespace: %v, status: %v\n", item.Name, item.Namespace, item.Status.Phase)

		} else {

```

Now a `--status` flag will provide additional output for the list.

```bash
go run cmd/kubectl-example/main.go  pod list --status
pod at-sample2-pod in namespace: default, status: Succeeded
pod foo in namespace: default, status: Running
```

Step 8. Using Rest Client

Lets add another pod list, but using the rest client.  Find the code `fmt.Printf("add pod list code using the rest client\n")` in `pod_list.go` and replace with the following:

First lets get a client, but in this case we are going to use the rest client.  Look in the `environment.go` file at the differences.

```go
	client := env.NewRestClient(&Settings)
	result := &v1.PodList{}
```

The REST API is more generic and it is coded using the builder pattern.  
```go
	err := client.Get().
		Namespace(apiv1.NamespaceDefault).
		Resource("pods").
		Do().
		Into(result)
	if err != nil {
		return err
	}
```

At this point the `PodList` object is the same and can be displayed the same way as in the first part of this lab.
```go
	if len(result.Items) == 0 {
		fmt.Printf("no pods discovered\n")
		return nil
	}
	for _, item := range result.Items {
			fmt.Fprintf(p.out, "pod %v in namespace: %v\n", item.Name, item.Namespace)
	}
```

Step 9: Adding a Pod

Up to this point in the lab, you have only queried Kubernetes for information.  In this part of the lab, you will add a pod programmically. You will be working with the `pod_add` file.  Find 
code that reads `fmt.Printf("adding a pod\n")` and replace with the following:

This step in the lab will require the following additions to the imports:

```go
	apiv1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

	"github.com/codementor/k8s-cli/pkg/example/env"
```

Similar to the first pod list, we need a client and a podClient.  

```go
	client := env.NewClientSet(&Settings)

	podsClient := client.CoreV1().Pods(apiv1.NamespaceDefault)
```

Next we need to define the pod using the v1.Pod API.  

```go
	pod := &apiv1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:   name,
			Labels: map[string]string{"app": "demo"},
		},
		Spec: apiv1.PodSpec{
			Containers: []apiv1.Container{
				{
					Name:  name,
					Image: p.image,
				},
			},
		},
	}
```

Notice the setting of `Name`, `Image` and `Labels`.  

Now lets create it!
```go
	pp, err := podsClient.Create(pod)
	if err != nil {
		return err
	}

	fmt.Fprintf(p.out, "Pod %v created with rev: %v\n", pp.Name, pp.ResourceVersion)
```

Notice we get another object back from create which contains updates to the object we passed.

Let's check the cluster `k get pods`.

