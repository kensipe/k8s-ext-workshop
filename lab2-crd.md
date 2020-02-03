# Lab Custom Resource Definition CRD

## Objective

The focus of this lab is to become familar with custom resource definitions (CRDs).  Through this lab you hand craft a CRD and add a new type to your kubernetes cluster.  The lab will include a look at openAPI v3 Schema with validation and kubectl explain support.

## Prerequisites

* Running Kubernetes 1.15+ cluster

1. Create CRD

You will be creating a new `kind` which is a Thermometer which can its unit's defined and is namespaced.

Example file named: `therm-crd.yaml`
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: thermometers.d2iq.com
spec:
  group: d2iq.com
  version: v1
  names:
    kind: Thermometer
    plural: thermometers
    shortNames:
      - therm
  scope: Namespaced
```

2. Installing a CRD

First, lets see what CRDs exist?

```
k get crds
No resources found in default namespace.
```

Without a defined CRD an error is returned if thermometers are queried.

```
k get therm
error: the server doesn't have a resource type "therm"
```

Lets install therm-crd...

`k apply -f therm-crd.yaml`

Now we have a 
```
k get crd
NAME                    CREATED AT
thermometers.d2iq.com   2020-02-02T05:35:47Z
```

resources are now successful...

```
k get therm
No resources found in default namespace.
```

api-resources now includes thermometers
```
k api-resources | grep therm
thermometers                      therm        d2iq.com                       true         Thermometer
```

which can be queried via the api 

`k get --raw /apis/d2iq.com/v1/thermometers`
```
{"apiVersion":"d2iq.com/v1","items":[],"kind":"ThermometerList","metadata":{"continue":"","resourceVersion":"9154","selfLink":"/apis/d2iq.com/v1/thermometers"}}
```
    
3. Let's create a Custom Resource (CR)

with a file named `stockholm.yaml`
```
apiVersion: d2iq.com/v1
kind: Thermometer
metadata:
    name: stockholm
    namespace: sweden
spec:
    unit: Celcius
```

Create namespace: `k create ns sweden`

Create CR: `k apply -f stockholm.yaml`

```
k get therm -A
NAMESPACE   NAME        AGE
sweden      stockholm   13s
```

It would be nice if the displayed output included more information regarding the resource.  Let's look at that.

4. Creating custom print columns

To the previous CRD (`therm-crd.yaml`) add the following:

```yaml
additionalPrinterColumns:
  - name: Unit
    type: string
    JSONPath: .spec.unit
  - name: Temperature
    type: string
    JSONPath: .status.temperature
```

Redeploy and retrieve the thermometers again:

```bash
# reapply crd
k apply -f therm-crd.yaml

# retrieve thermometers
k get therm -A
NAMESPACE   NAME        UNIT      TEMPERATURE
sweden      stockholm   Celcius 
```

5. Validation

First, Let's try a new CR such as:

```yaml
apiVersion: d2iq.com/v1
kind: Thermometer
metadata:
    name: gothenburg
    namespace: sweden
spec:
    unit: Celcius
    foo: test
```

Does it succesfully apply?

```bash
# apply
k apply -f gothenburg.yaml 
thermometer.d2iq.com/gothenburg created

# get therm
k get therm -A
NAMESPACE   NAME         UNIT      TEMPERATURE
sweden      gothenburg   Celcius   
sweden      stockholm    Celcius 
```

Now lets add a schema to the CRD using open API v3 Schema.  Add the following to the CRD.

```yaml
  validation:
      openAPIV3Schema:
        type: object
        description: Thermometer is the Schema for the Thermometer API.
        properties:
          apiVersion:
            type: string
          kind:
            type: string
          metadata:
            type: object
          spec:
            description: Thermometer Spec defines the desired state of Thermometer
            type: object
            properties:
              unit:
                description: Units in Celcius or Fahrenheit
                type: string
                anyOf: [{"pattern": "^Celcius"}, {"pattern": "^Fahrenheit"}]
            required: ["unit"]
          status:
            description: Thermometer Status defines the possible status states for Thermometer
            type: object
            properties:
              temperature:
                type: number
```

Delete the crds from the cluster `k delete crd --all` and reapply the crd. `k apply -f therm-crd.yaml`.

Try reapplying the gothenburg CR.  `k apply -f gothenburg.yaml`
Try stockholm manifest again.

6. Explaining CRDs

In the CRD manifest, define the following just after `scope: Namespaced`

```yaml
  scope: Namespaced
  preserveUnknownFields: false
```

The `preserveUnknownFields` is not needed for CRD v1, but it is needed for v1beta1.  The combination of this field set to false AND the defined schema enables the `k explain` such as:

```bash
k explain therm --recursive
KIND:     Thermometer
VERSION:  d2iq.com/v1

DESCRIPTION:
     Thermometer is the Schema for the Thermometer API.

FIELDS:
   apiVersion	<string>
   kind	<string>
   metadata	<Object>
      annotations	<map[string]string>
      clusterName	<string>
      creationTimestamp	<string>
      deletionGracePeriodSeconds	<integer>
      deletionTimestamp	<string>
      finalizers	<[]string>
      generateName	<string>
      generation	<integer>
      labels	<map[string]string>
      managedFields	<[]Object>
         apiVersion	<string>
         fieldsType	<string>
         fieldsV1	<map[string]>
         manager	<string>
         operation	<string>
         time	<string>
      name	<string>
      namespace	<string>
      ownerReferences	<[]Object>
         apiVersion	<string>
         blockOwnerDeletion	<boolean>
         controller	<boolean>
         kind	<string>
         name	<string>
         uid	<string>
      resourceVersion	<string>
      selfLink	<string>
      uid	<string>
   spec	<Object>
      unit	<string>
   status	<Object>
      temperature	<number>
```

## Summary

Custom Resource Defintion (CRDs) is the mechanism used to add new `kind`s into a kubernetes cluster.  It makes the kubernetes API extensible.  When added with controllers watching the CRDs, enables a custom declarative experience.