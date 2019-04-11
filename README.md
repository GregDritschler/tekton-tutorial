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
* A **Pipeline** defines the set of *Tasks* that compose a pipeline.
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

We will work from the bottom-up, i.e. first we will define the *Task* resources needed to build and deploy the image,
then we'll define the *Pipeline* resource that references the tasks,
and finally we'll create the *PipelineRun* and *PipelineResource* resources needed to run the pipeline.


### Create a task to build an image and push it to a container registry

Below is a Tekton task that builds a docker image and pushes it to a container registry.
You can find this yaml file at [tekton/tasks/source-to-image.yaml](tekton/tasks/source-to-image.yaml) file.

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

The task requires two PipelineResources:
* An input PipelineResource of type `git-source` that defines where the source is located
* An output PipelineResource of type `image` that defines the registry to which the image is pushed

Note that PipelineResources are arguments to the task.
They will be passed down to the Task from the higher-level resources PipelineRun and Pipeline which we'll see later.
This makes the task reusable with different git repositories and image registries.

A task also can have input parameters.  Parameters help to make a Task more reusable.
This task accepts three optional parameters:
* a path to the Docker build context inside the git source
* a path to the Dockerfile inside the build context
* an image tag to apply to the built image

Apply the file to your cluster to create the task.

```
kubectl apply -f tekton/tasks/source-to-image.yaml
```


## Create a task to deploy an image to a Kubernetes cluster

Below is a Tekton task that deploys a docker image to a Kubernetes cluster.
You can find this yaml file at [tekton/tasks/deploy-using-kubectl.yaml](tekton/tasks/deploy-using-kubectl.yaml) file.

```

```