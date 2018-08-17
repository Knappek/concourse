# Concourse

Here we document everything related to Concourse on Kubernetes. We will document how to provision Concourse on Kubernetes and then we will show how to implement an example pipeline
that builds a spring boot hello world with maven.

## Prerequisites
* running Kubernetes cluster (documentation will come!)
* install the [CLI tool `fly`](https://concourse-ci.org/download.html)

## Installation

The easiest way to install Concourse is with the [official stable Helm chart](https://github.com/kubernetes/charts/tree/master/stable/concourse).
For this you have to first initialize Helm:

1. Create a yaml file `cluster-rbac.yml` for cluster wide permissions for Helm (this can be changed and restricted to only certain namespaces)

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: tiller
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: tiller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: tiller
        namespace: kube-system
    ```
2. create the service account with the clusterrolebinding by applying the yaml

    ```yaml
    kubectl apply -f cluster-rbac.yml
    ```
3. initialize Helm (create Tiller)

    ```shell
    helm init --service-account tiller
    ```


Afterwards you can install the Concourse Helm Chart with

```shell
helm install --name concourse-app --namespace concourse stable/concourse -f concourse/values.yaml
```

where we have adapted the `values.yaml` a little bit to use an nginx ingress controller.

## Getting Started

There is a nice [Introduction to Concourse documentation](https://concoursetutorial.com/) with some getting started guides.

## Connect to Concourse

With `fly` login to your concourse instance:

```shell
fly --target <my-target> login --concourse-url http://<concourse-url>
```

where the `target` is something you choose, it is basically a local namespace to which you can refer. This is useful if you work with multiple concourse instances.

### Simple Task example

A very neat hello world: https://concoursetutorial.com/basics/task-hello-world/ 

This just executes a single task, but we may be interested in pipelines. As Concourse states "1% of tasks that Concourse runs are via fly execute. 99% of tasks that Concourse runs are within pipelines".

### Pipelines

There is a [basic pipeline example](https://concoursetutorial.com/basics/basic-pipeline/) but we want to outline a more sophisticated pipeline here.

Our pipeline basically consists of two files: the `pipeline.yaml`, which is the pipeline configuration in general (it does not need to live near the actual application code), 
and the `task_hello_world.yml`, which contains a task that should be executed and it lives near the application code. In the future, we would have multiple task yamls for one pipeline
in order to have a clean structure.

#### Create the pipeline

From now on we assume that we have logged in via fly with the target `andy-test` and our Concourse Web service is running on http://concourse.knappster.de.

##### Configure Pipeline

To create the pipeline, simply execute

```shell
fly set-pipeline -t andy-test -c material/pipeline.yaml -p hello-world-git
```

which will create a pipeline named `hello-world-git`.

Let's explore that file: 

The pipeline will be configured to be connected with the git repository `https://github.com/Knappek/spring-boot-hello-world-example.git` and uses the master branch. 
Furthermore a `webhook_token` is defined that can be used in github to create a github webhook. This is configured in the resource `resource-tutorial` (you can define more than one resource).

##### Configure Github Webhook

The pipeline has one job named `job-hello-world-git` which uses the above defined resource `resource-tutorial` that can be triggered (via the github webhook defined above - if you don't configure the webhook above, the pipeline will poll the git repo for changes once a minute, the webhook makes that trigger event based). The Job has one task whose task definition is defined in the file `task_hello_world.yml` that lives in the resource `resource-tutorial`, i.e. in the git repo. 

The next thing what we need to do is to configure the webhook at Github: Navigate to your github repository in the browser to `Settings > Webhooks` and paste the webhook url `http://concourse.knappster.de/api/v1/teams/main/pipelines/hello-world-git/resources/resource-tutorial/check/webhook?webhook_token=<webhook_token>` to the field `Paylod URL`. Leave everything else as default and save it.

##### Create task definition

Last thing what we need to do is to create our task definition. A simple task definition, which we name `task_hello_world.yml` can look like

```yaml
---
platform: linux

image_resource:
  type: docker-image
  source: {repository: busybox}

run:
  path: cat
  args: [resource-tutorial/task_hello_world.yml]
inputs:
  - name: resource-tutorial
```

which basically only prints the output of itself. This task is running in one of the Concourse worker (pods) by using the docker image `busybox` (which is a lightweight linux Docker image without special functionality). The `inputs` section defines to actually use the resource that is defined in the `pipeline.yaml`, i.e. cloning the git repo.

Push that file into your Github Repository (defined in the `pipeline.yaml`) and push it. That's it, you have your first concourse pipeline running, everything defined as code declaratively in yaml.

