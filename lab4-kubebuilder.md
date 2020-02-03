# Lab Kubebuilder: Creating a CRD and Controller

## Objective

The focus of this lab is to become familar with kubebuilder.  In this lab you will create a CRD through go structs and automation.  You will see how RBACs are created through generation from code annotations.  Then you will create a controller for working with the CRD.

This lab and code was inspired by the previous work of https://github.com/programming-kubernetes/cnat and in particular https://github.com/programming-kubernetes/cnat/tree/master/cnat-kubebuilder, however that code is out of date for the current versions of controller-runtime and kubebuilder.  Using it as a reference you will notice some nice improvements to standard patterns in controller-runtime. 

The basics of this operator is that we will have an `at` CRD which takes a time and command.  A controller will monitor the CRD and Pods.  The controller will create `at` kinds and will create a pod to run the command if the time has past.  Once the pod is finished running, the controller will update the CR regarding the pod status.

## Prerequisites

* Running Kubernetes 1.15+ cluster
* kubebuilder 2.20 ([install from site](https://github.com/kubernetes-sigs/kubebuilder#installation)) or `brew install kubebuilder`
* go 1.13+

1. Creating a Go project

```bash
$GOPATH = /Users/kensipe/projects/go
mkdir $GOPATH/src/github.com/kensipe/at-controller
cd $GOPATH/src/github.com/kensipe/at-controller

go mod init github.com/kensipe/at-controller
```

2. Initialize kubebuilder

`kubebuilder init --domain d2iq.com --owner “jfokus"`

Review the create structure and files.  In particular `main.go`.  The project has started but there isn't much there yet.

3. Add an API

```bash
kubebuilder create api \
 --group cnat \
 --version v1alpha1 \
 --kind At
```

Now it's time to review a number of files. `main.go` has changed.  Review `at_types.go` and `at_controller.go`.

4. Defining the CRD via Go struts

Start with `at_types.go`.  We want to change the `Spec` and `Status` similar to the CRD lab.  This requires changes to `AtSpec` and `AtStatus` respectively.  You'll notice a defined `Foo` in Spec which should be removed.   

For `AtSpec` add `Schedule` and `Command` both are strings.  You will need the `json...` annotation and can use the generated Foo as an example.

For `AtStatus` you need to add a string variable named `Phase`.

To complete the types definition and for controller convenience define the following phases in the `at_types.go` file.

```go
const (
	PhasePending = "PENDING"
	PhaseRunning = "RUNNING"
	PhaseDone    = "DONE"
)
```

Let's try it out...  In the terminal from the project root run: `make manifests` which will generate files in the `config` folder AND/OR `make install` which will apply the CRDs to the running kubernetes cluster.

```bash
# after make install the CRD is installed
 k get crd
NAME                    CREATED AT
ats.cnat.d2iq.com       2020-02-02T09:48:23Z
```

Let's create a CR from this CRD...

```yaml
apiVersion: cnat.d2iq.com/v1alpha1
kind: At
metadata:
  name: at-sample2
spec:
  schedule: "2020-01-30T10:02:00Z"
  command: "echo YAY"
```


**advanced:** Looking to add the printer columns?  The following build tags can be placed before `type At struct`

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:JSONPath=".spec.schedule", name=Schedule, type=string
// +kubebuilder:printcolumn:JSONPath=".status.phase", name=Phase, type=string
// At is the Schema for the ats API
type At struct {
	metav1.TypeMeta   `json:",inline"`

```

Reinstall manifests with `make install`

Retrieve the CR

```bash
k get at
NAME         SCHEDULE               PHASE
at-sample2   2020-01-30T10:02:00Z   
```

----
Now that we have a CRD to work with lets focus on the controller.

in the `at_controller.go` file, there are 2 tags to generate RBAC for the CRD, however we this controller will need permission for pods as well.  

find:

```go
// +kubebuilder:rbac:groups=cnat.d2iq.com,resources=ats,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cnat.d2iq.com,resources=ats/status,verbs=get;update;patch
```

and add:

```go
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps,resources=deployments/status,verbs=get;update;patch
```
For more details on kubebuilder markers read: https://book.kubebuilder.io/reference/markers.html


Working on the `func (r *AtReconciler) Reconcile` function, change the logger to a specific logger name and with some defined structure as follows:

```go
	logger := r.Log.WithValues("namespace", req.NamespacedName, "at", req.Name)
	logger.Info("== Reconciling At")
```

Following the logger, is a good place for fetching instances of the CR for `at` as follows:

```go
	// Fetch the At instance
	instance := &cnatv1alpha1.At{}
	err := r.Get(context.TODO(), req.NamespacedName, instance)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request - return and don't requeue:
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request:
		return reconcile.Result{}, err
	}
```

Now that we have an instance defined by the request namespacedname, lets check to see if it has a status, if not, lets initialize.

```go
	// If no phase set, default to pending (the initial phase):
	if instance.Status.Phase == "" {
		instance.Status.Phase = cnatv1alpha1.PhasePending
		cnatv1alpha1.PhasePending)
	}
```

While there is some additional logic you will want to add for working an instance through it's phases, lets follow this up with an update which will define the end of our function just prior to a return.

```go
	// Update the At instance, setting the status to the respective phase:
	err = r.Status().Update(context.TODO(), instance)
	if err != nil {
		return reconcile.Result{}, err
	}

	return ctrl.Result{}, nil
```

It is now possible to see some work within this controller.  You will need to re-run `make install` to setup new RBAC manifests.  Then to run the controller.. run the following:

`make run`

If you have an instance of `at`, after running the controller the status should have been updated... try `k get at`

Completing the controller requires a couple of support functions for creating the pod and checking the schedule.  Add the following functions to the `at_controller.go` file.

```go
// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *cnatv1alpha1.At) *corev1.Pod {
	labels := map[string]string{
		"app": cr.Name,
	}
	return &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      cr.Name + "-pod",
			Namespace: cr.Namespace,
			Labels:    labels,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "busybox",
					Image:   "busybox",
					Command: strings.Split(cr.Spec.Command, " "),
				},
			},
			RestartPolicy: corev1.RestartPolicyOnFailure,
		},
	}
}

// timeUntilSchedule parses the schedule string and returns the time until the schedule.
// When it is overdue, the duration is negative.
func timeUntilSchedule(schedule string) (time.Duration, error) {
	now := time.Now().UTC()
	layout := "2006-01-02T15:04:05Z"
	s, err := time.Parse(layout, schedule)
	if err != nil {
		return time.Duration(0), err
	}
	return s.Sub(now), nil
}
```

Finishing the `Reconcile` function, insert the code below after the if body that sets the `instance.Status.Phase = cnatv1alpha1.PhasePending`:

```go
// Now let's make the main case distinction: implementing
	// the state diagram PENDING -> RUNNING -> DONE
	switch instance.Status.Phase {
	case cnatv1alpha1.PhasePending:
		logger.Info("Phase: PENDING")
		// As long as we haven't executed the command yet, we need to check if it's time already to act:
		logger.Info("Checking schedule", "Target", instance.Spec.Schedule)
		// Check if it's already time to execute the command with a tolerance of 2 seconds:
		d, err := timeUntilSchedule(instance.Spec.Schedule)
		if err != nil {
			logger.Error(err, "Schedule parsing failure")
			// Error reading the schedule. Wait until it is fixed.
			return reconcile.Result{}, err
		}
		logger.Info("Schedule parsing done", "Result", fmt.Sprintf("diff=%v", d))
		if d > 0 {
			// Not yet time to execute the command, wait until the scheduled time
			return reconcile.Result{RequeueAfter: d}, nil
		}
		logger.Info("It's time!", "Ready to execute", instance.Spec.Command)
		instance.Status.Phase = cnatv1alpha1.PhaseRunning
	case cnatv1alpha1.PhaseRunning:
		logger.Info("Phase: RUNNING")
		pod := newPodForCR(instance)
		// Set At instance as the owner and controller
		if err := controllerutil.SetControllerReference(instance, pod, r.Scheme); err != nil {
			// requeue with error
			return reconcile.Result{}, err
		}
		found := &corev1.Pod{}
		err = r.Get(context.TODO(), types.NamespacedName{Name: pod.Name, Namespace: pod.Namespace}, found)
		// Try to see if the pod already exists and if not
		// (which we expect) then create a one-shot pod as per spec:
		if err != nil && errors.IsNotFound(err) {
			err = r.Create(context.TODO(), pod)
			if err != nil {
				// requeue with error
				return reconcile.Result{}, err
			}
			logger.Info("Pod launched", "name", pod.Name)
		} else if err != nil {
			// requeue with error
			return reconcile.Result{}, err
		} else if found.Status.Phase == corev1.PodFailed || found.Status.Phase == corev1.PodSucceeded {
			logger.Info("Container terminated", "reason", found.Status.Reason, "message", found.Status.Message)
			instance.Status.Phase = cnatv1alpha1.PhaseDone
		} else {
			// don't requeue because it will happen automatically when the pod status changes
			return reconcile.Result{}, nil
		}
	case cnatv1alpha1.PhaseDone:
		logger.Info("Phase: DONE")
		return reconcile.Result{}, nil
	default:
		logger.Info("NOP")
		return reconcile.Result{}, nil
	}
```

If you run the controller again `make run` you should see phase status changing for the CR but it never fully gets to "Done".  This is because the controller isn't watching pods yet.

The final modification needed is in the `SetupWithManager` function.  Make the following changes:

```go
	return ctrl.NewControllerManagedBy(mgr).
		For(&cnatv1alpha1.At{}).
		Owns(&cnatv1alpha1.At{}).
		Owns(&corev1.Pod{}).
		Complete(r)
```

---
**Advanced:** Using the kubernete events

Take a look a the desription of the at RC using `k describe at at-sample`

```bash
 k describe at at-sample
Name:         at-sample
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"cnat.d2iq.com/v1alpha1","kind":"At","metadata":{"annotations":{},"name":"at-sample","namespace":"default"},"spec":{"comman...
API Version:  cnat.d2iq.com/v1alpha1
Kind:         At
Metadata:
  Creation Timestamp:  2020-02-02T11:30:46Z
  Generation:          1
  Resource Version:    27790
  Self Link:           /apis/cnat.d2iq.com/v1alpha1/namespaces/default/ats/at-sample
  UID:                 89d3ef30-8714-47b9-bbf8-664262b00610
Spec:
  Command:   echo YAY
  Schedule:  2020-01-30T10:02:00Z
Status:
  Phase:  RUNNING
```

Notice there are no "events" against this object.  This step of the lab changes that.

Add the `Recorder record.EventRecorder` to the `AtReconciler` struct so that it looks like:

```go
// AtReconciler reconciles a At object
type AtReconciler struct {
	client.Client
	Log    logr.Logger
	Scheme *runtime.Scheme
	Recorder record.EventRecorder
}
```

This struct is initialized in `main.go`,  Modify this file to the following:

```go
if err = (&controllers.AtReconciler{
		Client:   mgr.GetClient(),
		Log:      ctrl.Log.WithName("controllers").WithName("At"),
		Scheme:   mgr.GetScheme(),
		Recorder: mgr.GetEventRecorderFor("at-controller"),	
```

Now modify the `at_controller.go` code to record the events for each transition of the phase status.  Below is an example of when the phase is set to "Pending"

```go
r.Recorder.Event(instance, "Normal", "PhaseChange", cnatv1alpha1.PhasePending)
```

The results of a describe after this modification will now look like:

```bash
k describe at at-sample`

```bash
 k describe at at-sample
Name:         at-sample
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"cnat.d2iq.com/v1alpha1","kind":"At","metadata":{"annotations":{},"name":"at-sample","namespace":"default"},"spec":{"comman...
API Version:  cnat.d2iq.com/v1alpha1
Kind:         At
Metadata:
  Creation Timestamp:  2020-02-02T11:30:46Z
  Generation:          1
  Resource Version:    27790
  Self Link:           /apis/cnat.d2iq.com/v1alpha1/namespaces/default/ats/at-sample
  UID:                 89d3ef30-8714-47b9-bbf8-664262b00610
Spec:
  Command:   echo YAY
  Schedule:  2020-01-30T10:02:00Z
Status:
  Phase:  RUNNING
Events:
  Type    Reason       Age   From           Message
  ----    ------       ----  ----           -------
  Normal  PhaseChange  34s   at-controller  PENDING
  Normal  PhaseChange  34s   at-controller  RUNNING
```

### Notes

The following are the imports needed for the `at_controller.go` for the changes indicated in this lab.

```go
import (
	"context"
	"fmt"
	"strings"
	"time"

	"github.com/go-logr/logr"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/tools/record"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"

	cnatv1alpha1 "github.com/codementor/cnat/api/v1alpha1"
)
```
