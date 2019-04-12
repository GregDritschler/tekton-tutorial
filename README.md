## Knative Deployment using Tekton Pipelines

Tekton is an open source project to configure and run CI/CD pipelines within a Kubernetes cluster.


## Objectives

In this tutorial you'll learn
* what are the basic concepts used by Tekton pipelines
* how to create a pipeline to build and deploy a Knative application
* how to run the pipeline, check its status and troubleshoot problems


## Prerequisites

* Create a standard Kubernetes cluster in IBM Kubernetes Service

* Create a private container registry in IBM Container Service

* Install Knative in your cluster

* Install Tekton in your cluster by following the instructions [here](https://cloud.google.com/tekton/)


## Tekton pipeline concepts

Tekton provides a set of extensions to Kubernetes, in the form of [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) for defining pipelines.
The following diagram shows the resources used in this tutorial.  The arrows depict references from one resource to another resource.

![crd](doc/source/images/crd.png)

The resources are used as follows.

* A **PipelineResource** defines an object that is an input (such as a git repository) or an output (such as a docker image) of the pipeline.
* A **PipelineRun** defines an execution of a pipeline.  It references the **Pipeline** to run and the **PipelineResources** to use as inputs and outputs.
* A **Pipeline** defines the set of **Tasks** that compose a pipeline.
* A **Task** defines a set of build steps such as compiling code, running tests, and building and deploying images.

We will go into more detail about each resource during the walkthrough of the example.


## Sample pipeline

Let's create a simple pipeline that

* builds a Docker image from source files and pushes it to your private container registry
* deploys the image as a Knative service in your Kubernetes cluster

You should clone this project to your workstation since you will need to edit some of the yaml files before applying them to your cluster.

```
git clone https://github.com/gregdritschler/tekton-tutorial
```

We will work from the bottom-up, i.e. first we will define the Task resources needed to build and deploy the image,
then we'll define the Pipeline resource that references the tasks,
and finally we'll create the PipelineRun and PipelineResource resources needed to run the pipeline.


### Create a task to build an image and push it to a container registry

Below is a Tekton task that builds a docker image and pushes it to a container registry.
You can find this yaml file at [tekton/tasks/source-to-image.yaml](tekton/tasks/source-to-image.yaml).

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: source-to-image
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToDockerFile
        description: The path to the dockerfile to build (relative to the context)
        default: Dockerfile
      - name: imageTag
        description: Tag to apply to the built image
        default: "latest"
  outputs:
    resources:
      - name: built-image
        type: image
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor
      command:
        - /kaniko/executor
      args:
        - --dockerfile=${inputs.params.pathToDockerFile}
        - --destination=${outputs.resources.built-image.url}:${inputs.params.imageTag}
        - --context=/workspace/git-source/${inputs.params.pathToContext}
```

A task can have one or more steps.  Each step defines an image to run to perform the function of the step.
This task has one step that uses the [kaniko](https://github.com/GoogleContainerTools/kaniko) project to build a docker image from source and push it to a registry.

The task requires two resources:
* An input resource of type `git-source` that defines where the source is located
* An output resource of type `image` that defines the registry to which the image is pushed

Note that the resources are simply abstract arguments to the task.
We'll see later how they become bound to PipelineResources which define the actual resource to be used.
This makes the task reusable with different git repositories and image registries.

A task also can have input parameters.  Parameters help to make a Task more reusable.
This task accepts three optional parameters:
* a path to the Docker build context inside the git source
* a path to the Dockerfile inside the build context
* an image tag to apply to the built image

You may be wondering about how the task authenticates to the image registry for permission to push the image.
This will be covered later on in the tutorial.

Apply the file to your cluster to create the task.

```
kubectl apply -f tekton/tasks/source-to-image.yaml
```


## Create a task to deploy an image to a Kubernetes cluster

Below is a Tekton task that deploys a docker image to a Kubernetes cluster.
You can find this yaml file at [tekton/tasks/deploy-using-kubectl.yaml](tekton/tasks/deploy-using-kubectl.yaml).

```
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: deploy-using-kubectl
spec:
  inputs:
    resources:
      - name: git-source
        type: git
      - name: built-image
        type: image
    params:
      - name: pathToYamlFile
        description: The path to the yaml file to deploy within the git source
      - name: imageTag
        description: Tag of the images to be used.
        default: "latest"
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;__IMAGE__;${inputs.resources.built-image.url}:${inputs.params.imageTag};g"
        - "/workspace/git-source/${inputs.params.pathToYamlFile}"
    - name: run-kubectl
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "/workspace/git-source/${inputs.params.pathToYamlFile}"
```

This task has two steps.

1. The first step runs `sed` in an Alpine Linux container to update the yaml file used for deployment with the image that was built by the source-to-image task.
The step requires the yaml file to have a character string `__IMAGE__` at the point where this update needs to occur.

2. The second step runs `kubectl` using Lachlan Evenson's popular `k8s-kubectl` container image to apply the yaml file to the same cluster where the pipeline is running.

As was the case in the source-to-image task, this task makes use of input PipelineResources and parameters in order to make the task as reusable as possible.

You may be wondering about how the task authenticates to the cluster for permission to apply the resource(s) in the yaml file.
This will be covered later on in the tutorial.

Apply the file to your cluster to create the task.

```
kubectl apply -f tekton/tasks/deploy-using-kubectl.yaml
```


## Create a pipeline

Below is a Tekton pipeline that runs the two tasks we defined above.
You can find this yaml file at [tekton/pipeline/build-and-deploy-pipeline.yaml](tekton/pipeline/build-and-deploy-pipeline.yaml).

```
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: build-and-deploy-pipeline
spec:
  resources:
    - name: git-source
      type: git
    - name: built-image
      type: image
  params:
    - name: pathToContext
      description: The path to the build context, used by Kaniko - within the workspace
      default: src
    - name: pathToYamlFile
      description: The path to the yaml file to deploy within the git source
    - name: imageTag
      description: Tag to apply to the built image
  tasks:
  - name: source-to-image
    taskRef:
      name: source-to-image
    params:
      - name: pathToContext
        value: "${params.pathToContext}"
      - name: imageTag
        value: "${params.imageTag}"
    resources:
      inputs:
        - name: git-source
          resource: git-source
      outputs:
        - name: built-image
          resource: built-image
  - name: deploy-to-cluster
    taskRef:
      name: deploy-using-kubectl
    params:
      - name: pathToYamlFile
        value:  "${params.pathToYamlFile}"
      - name: imageTag
        value: "${params.imageTag}"
    resources:
      inputs:
        - name: git-source
          resource: git-source
        - name: built-image
          resource: built-image
          from:
            - source-to-image
```

A Pipeline resource lists the tasks to run and provides the input and output resources and input parameters required by each task.
All resources must be exposed as inputs or outputs of the pipeline;  the pipeline cannot bind one to an actual PipelineResource.
However you can choose whether to expose a task's input parameter as a pipeline input parameter, set the value directly, or let the value
default inside the task (if it's an optional parameter).  For example this pipeline exposes the `pathToContext` parameter from the
source-to-image task but does not expose the `pathToDockerFile` parameter and allows it to default inside the task.

Dependencies between tasks can be expressed by using the `from` key with an input resource.
The value specifies one or more tasks which output the resource and must run first.
In this example, the pipeline specifies that the `built-image` resource used in the `deploy-using-kubectl` task comes `from` the `source-to-image` task.
Tekton will order to the execution of the tasks to satisfy this dependency.

Dependencies between tasks also can be expressed by using the `runAfter` key which isn't used in this tutorial
but which you can [read about in the official Tekton documentation]((https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md#runafter).

Apply the file to your cluster to create the pipeline.

```
kubectl apply -f tekton/pipeline/build-and-deploy-pipeline.yaml
```


## Create PipelineRun and PipelineResources

We've defined reusable Pipeline and Task resources for building and deploying an image.
It is now time to look at how one runs the pipeline with an actual set of input and output resources.

Below is a Tekton PipelineRun resource that runs the pipeline we defined above.
You can find this yaml file at [tekton/run/picalc-pipeline-run.yaml](tekton/pipeline/picalc-pipeline-run.yaml).

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  generateName: picalc-pr-
spec:
  pipelineRef:
    name: build-and-deploy-pipeline
  resources:
    - name: git-source
      resourceRef:
        name: picalc-git
    - name: built-image
      resourceRef:
        name: picalc-image
  params:
    - name: pathToYamlFile
      value: "knative/picalc.yaml"
    - name: imageTag
      value: "1.0"
  trigger:
    type: manual
  serviceAccount: pipeline-account
```

Although this file is small there is a lot going on here.  Let's break it down from top to bottom:

* The PipelineRun does not have a fixed name.  It uses `generateName` to generate a name each time it is created.
This is because a particular PipelineRun resource executes the pipeline only once.  If you want to run the pipeline again,
you cannot modify an existing PipelineRun resource to request it to re-run -- you must create a new PipelineRun resource.
While you could use `name` to assign a unique name to your PipelineRun each time you create one, it is much easier to use `generateName`.

* The Pipeline resource is identified under the `pipelineRef` key.

* The resources required by the pipeline are bound to specific PipelineResources named `picalc-git` and `picalc-image`.
We will define these in a moment.

* Parameters exposed by the pipeline are set to specific values.

* A service account named `pipeline-account` is specified to provide the credentials needed for the pipeline to run successfully.
We will define this service account in the next part of the tutorial.

Below is the Tekton PipelineResource for `picalc-git` which defines the git source.
You can find this yaml file at [tekton/resources/picalc-git.yaml](tekton/resources/picalc-git.yaml).

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: picalc-git
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/IBM/tekton-tutorial
```

The source code for this example is a [go program that calculates an approximation of pi](src/picalc.go).
The source includes a [Dockerfile](src/Dockerfile) which runs tests, compiles the code, and builds an image for execution.

Below is the Tekton PipelineResource for `picalc-image` which defines the docker image.
You can find this yaml file at [tekton/resources/picalc-image.yaml](tekton/resources/picalc-image.yaml).

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: picalc-image
spec:
  type: image
  params:
    - name: url
      description: The target URL
      value: <REGISTRY>/<NAMESPACE>/picalc
```

You must edit this file before applying it to substitute the values of <REGISTRY> and <NAMESPACE> with the information for your private container registry.

* To find the value for <REGISTRY>, enter the command `ibmcloud cr region`.
* To find the value of <NAMESPACE>, enter the command `ibmcloud cr namespace-list`.

Apply the two PipelineResource files to your cluster.
Do not apply the PipelineRun file yet because we still need to define the service account for it.

```
kubectl apply -f tekton/pipeline/tekton/resources/picalc-git.yaml
kubectl apply -f tekton/pipeline/tekton/resources/picalc-image.yaml
```

## Define a service account

The last step before running the pipeline is to set up a service account so that it can access protected resources.
The service account ties together a couple of secrets containing credentials for authentication
along with RBAC-related resources for permission to create and modify certain Kubernetes resources.

First you need to enable programmatic access to your private container registry by creating either
a registry token or an IBM Cloud Identity and Access Management (IAM) API key.
The process for creating a token or an API key is described [here](https://cloud.ibm.com/docs/services/Registry?topic=registry-registry_access#registry_access).

After you have the token or API key, you can create the following secret.

```
kubectl create secret generic ibm-cr-push-secret --type="kubernetes.io/basic-auth" --from-literal=username=<USER> --from-literal-password=<TOKEN/APIKEY>
kubectl annotate secret ibm-cr-push-secret tekton.dev/docker-0=<REGISTRY>
```

where

* `<USER>` is either `token` if you are using a token or `iamapikey` if you are using an API key
* `<TOKEN/APIKEY>` is either the token or API key that you created
* `<REGISTRY>` is the URL of your container registry, such as `us.icr.io` or `registry.ng.bluemix.net`

Now you can create the service account using the following yaml.
You can find this yaml file at [tekton/pipeline-account.yaml](tekton/pipeline-account.yaml).

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-account
secrets:
- name: ibm-cr-push-secret

---

apiVersion: v1
kind: Secret
metadata:
  name: kube-api-secret
  annotations:
    kubernetes.io/service-account.name: pipeline-account
type: kubernetes.io/service-account-token

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pipeline-role
rules:
- apiGroups: ["serving.knative.dev"]
  resources: ["services"]
  verbs: ["get", "create", "update", "patch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pipeline-role
subjects:
- kind: ServiceAccount
  name: pipeline-account
```

This yaml creates the following Kubernetes resources:

* A ServiceAccount named `pipeline-account`. This is the name that the PipelineRun seen earlier uses to reference this account.
The service account references the `ibm-cr-push-secret` secret so that the pipeline can authenticate to your private container registry
when it pushes a container image.

* A Secret named `kube-api-secret` which contains an API credential (generated by Kubernetes) for accessing the Kubernetes API.
This allows the pipeline to use `kubectl` to talk to your cluster.

* A Role named `pipeline-role` and a RoleBinding named `pipeline-role-binding` which provide the resource-based access control permissions
needed for this pipeline to create and modify Knative services.

Apply the file to your cluster to create the service account and related resources.

```
kubectl apply -f tekton/pipeline-account.yaml
```


## Run the pipeline

All the pieces are in place to run the pipeline.

```
kubectl create -f tekton/run/picalc-pipeline-run.yaml
```

Note that we're using `create` here instead of `apply`.
As mentioned previously a given PipelineRun resource can run a pipeline only once so you need to create a new one each time you want to run the pipeline.
`kubectl` will respond with the generated name of the PipelineRun resource.

```
pipelinerun.tekton.dev/picalc-pr-qhl5r created
```

You can check that status of the pipeline using the `kubectl describe` command:

```
kubectl describe pipelinerun picalc-pr-qhl5r
```

If you enter the command relatively quickly after creating the PipelineRun, you are most likely to see output similar to this:

```
Name:         picalc-pr-qhl5r
Namespace:    default
Labels:       tekton.dev/pipeline=build-and-deploy-pipeline
Annotations:  <none>
API Version:  tekton.dev/v1alpha1
Kind:         PipelineRun
Metadata:
  Creation Timestamp:  2019-04-12T19:53:27Z
  Generate Name:       picalc-pr-
  Generation:          1
  Resource Version:    3373283
  Self Link:           /apis/tekton.dev/v1alpha1/namespaces/default/pipelineruns/picalc-pr-qhl5r
  UID:                 a37dd088-5d5c-11e9-92d6-06ff308aa774
Spec:
  Status:  
  Params:
    Name:   pathToYamlFile
    Value:  knative/picalc.yaml
    Name:   imageTag
    Value:  1.0
  Pipeline Ref:
    Name:  build-and-deploy-pipeline
  Resources:
    Name:  git-source
    Resource Ref:
      Name:  picalc-git
    Name:    built-image
    Resource Ref:
      Name:         picalc-image
  Service Account:  pipeline-account
  Trigger:
    Type:  manual
Status:
  Conditions:
    Last Transition Time:  2019-04-12T19:53:27Z
    Message:               Not all Tasks in the Pipeline have finished executing
    Reason:                Running
    Status:                Unknown
    Type:                  Succeeded
  Start Time:              2019-04-12T19:53:27Z
  Task Runs:
    Picalc - Pr - Qhl 5 R - Source - To - Image - Mfc 5 Z:
      Pipeline Task Name:  source-to-image
      Status:
        Conditions:
          Last Transition Time:  2019-04-12T19:53:28Z
          Message:               pod status "PodScheduled":"False"; message: "pod has unbound immediate PersistentVolumeClaims (repeated 2 times)"
          Reason:                Pending
          Status:                Unknown
          Type:                  Succeeded
        Pod Name:                picalc-pr-qhl5r-source-to-image-mfc5z-pod-666336
        Start Time:              2019-04-12T19:53:27Z
Events:                          <none>

```

Note the status message for the source-to-image task says 

```
pod status "PodScheduled":"False"; message: "pod has unbound immediate PersistentVolumeClaims (repeated 2 times)"
```

Each PipelineRun creates a PersistentVolumeClaim requesting a persistent volume.
This volume is intended to be used to pass resources between tasks in the pipeline but the functionality is not fully implemented in Tekton at the time of this writing.
The persistent volume is [dynamically provisioned by IBM Cloud](https://cloud.ibm.com/docs/containers?topic=containers-kube_concepts#dynamic_provisioning).
This may take a minute or two.

Continue to check the status.  If the pipeline runs successfully, the overall status should look like this:

```
Status:
  Conditions:
    Last Transition Time:  2019-04-12T19:57:30Z
    Message:               All Tasks have completed executing
    Reason:                Succeeded
    Status:                True
    Type:                  Succeeded
  Start Time:              2019-04-12T19:53:27Z
```

Check the status of the deployed Knative service.  It should be ready.

```
$ kubectl get ksvc picalc
NAME      DOMAIN                                                          LATESTCREATED   LATESTREADY    READY     REASON
picalc    picalc.default.mycluster6.us-south.containers.appdomain.cloud   picalc-00001    picalc-00001   True
```

You can use the URL in the response to curl the service.

```
$ curl picalc.default.mycluster6.us-south.containers.appdomain.cloud?iterations=20000000
3.1415926036
```

If the pipeline did not run successfully, the overall status may look like this:

```
Status:
  Conditions:
    Last Transition Time:  2019-04-12T19:57:30Z
    Message:               TaskRun picalc-pr-qhl5r-deploy-to-cluster-7h8pm has failed
    Reason:                Failed
    Status:                False
    Type:                  Succeeded
  Start Time:              2019-04-12T19:53:27Z
```

Under the task run status you should find a message that tells you how to get the logs from the failed build step.

```
build step "build-step-deploy-using-kubectl" exited with code 1 (image: "docker.io/library/alpine@sha256:28ef97b8686a0b5399129e9b763d5b7e5ff03576aa5580d6f4182a49c5fe1913"); for logs run: kubectl -n default logs picalc-pr-qhl5r-deploy-to-cluster-7h8pm-pod-582c73 -c build-step-deploy-using-kubectl
```

You should delete both successful and failed PipelineRun resources when you are done with them to reclaim resources, especially the persistent volume associated with each one.

```
kubectl delete pipelinerun picalc-pr-qhl5r
```

