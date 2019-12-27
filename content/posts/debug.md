---
title: "Debugging Tekton Pipelines"
date: 2019-12-27T14:42:42+05:30
featured_image: '/images/debug.jpeg'
draft: false
---
## Tekton Pipeline

### About

**Tekton Pipelines** is a cloud-native CI/CD solutions. It provides Kubernetes style resources to define CI/CD pipelines and is designed to meet the CI/CD needs of both microservices and monolithic applications. If you want to know more about Tekton Pipelines, you can go through this [blog](https://medium.com/@nikhilthomas1/cloud-native-cicd-on-openshift-with-openshift-pipelines-tektoncd-pipelines-e944cd10341a7)

In this blog, we will cover some basic Tekton Pipelines concepts and how to debug a pipeline in case of failure.

### Pipeline Concepts

**Tekton Pipelines** leverages Custom Resource Definition (CRD) feature provided by Kubernetes to add CI/CD functionality. It uses Custom Resources as building blocks to define a CI/CD pipeline. This lets users interact with these custom resources just like any other Kubernetes resource.

Tekton Pipelines adds with the following custom resources:

 - **Task**: A task defines the number of steps that run sequentially to perform a specific action/task. They are defined once and can be shared by multiple Pipelines.
    * A Step, is a basic unit of a Task. For example, executing a command in a container spun up with a defined configuration for image, volumes, environment variables etc.
 - **Pipeline**: A Pipeline defines a workflow or an order of execution for Tasks along with the inputs and the outputs 
 - **PipelineResource**: These are used to define the Inputs and Outputs to a Pipeline or a Task.  
 You can define the following PipelineResources:
    * Source-code (pull request or git repository with branch or revision)
    * Image
    * Storage bucket
    * Cluster to use in Pipeline
    * Cloud Event
 - **TaskRun**: An invocation or an instantiation of a Task with its Inputs and Outputs.
 - **PipelineRun**: An invocation of a Pipeline with Inputs and Outputs.

A visual representation of the Pipeline concepts.

{{< figure src="/images/pipeline.jpg" >}}

A Pipeline consists of a set of Tasks that executes in a certain order with optional Inputs and Outputs. A Task further contains one or more steps that gets executed sequentially with the required resources. Tasks and Resources are reusable definitions that can be used by multiple pipelines. Now, in order to execute a Pipeline, you need to create a PipelineRun. A Task can also be executed individually by creating a TaskRun.

### Behind the scenes

When you create a PipelineRun to execute a Pipeline, it creates TaskRuns for all the Tasks defined in the Pipeline. To execute the Task which has steps defined as container specification, the TaskRun spins up a pod with one container for each step with the provided configuration and executes the command specified as the entrypoint in that container.

### Debugging Tekton Pipeline failures

To find the status of PipelineRun or a Taskrun, check the logs of the PipelineRun, find the TaskRuns of a PipelineRun, find the respective pods, see the logs, debug in case of failure.

 - Get the status of Pipeline/Taskrun

   ```shell script
   kubectl get pipelinerun <pipelinerun-name>
   ```

   or

   ```shell script
   kubectl get taskrun <taskrun-name>
   ```

   You will find a column **SUCCEEDED** having value true or false

 - Get the logs of PipelineRun/TaskRun

    1. This is where the magic of labels comes in :
       * When you create a PipelineRun to execute a given Pipeline
           1. A **tekton.dev/pipeline=pipelineName** label gets added to the PipelineRun, all TaskRuns and the pods.
           2. A **tekton.dev/pipelineRun=pipelineRunName** label gets added to all TaskRuns and pods.
       * Similarly when you create a TaskRun to execute a given Task.
           1. A **tekton.dev/task=taskName** label gets added to the TaskRun and the pod.
           2. A **tekton.dev/taskRun=taskRunName** label gets added to the pod.
    2. Thus, Pods associated with a PipelineRun can be found by:
       
       ```shell script
       kubectl get pods -l tekton.dev/pipelineRun=test-pipelinerun
       ```
       
       Pods associated with a task using: 
       
       ```shell script
       kubectl get pods -l tekton.dev/task=test-task
       ```

   3. If you have defined the step with the name **build**, the container name in the pod will be prefixed with the step keyword, in this case, **step-build**. This allows for logs to be fetched using 

      ```shell script
      kubectl logs <podname> -c <containername>
      ```

 - Analyzing a PipelineRun status

    1. PipelineRun status structure overview

       You can fetch the status of a PipelineRun using:

       ```shell script
       kubectl get pipelinerun <pipelinerun-name> -o yaml
       ```

       The structure of a PipelineRun status will look something like this:
       ```
       Status
           Conditions
           -   Message : (Error msg in case of failure)
               Reason : It is status of pipelinerun (Succeeded or Failed)
       
           Taskruns
           -   Taskrunname (Taskrun created)
               -   Pipelinetaskname - (name of the task in pipeline)
                   Status:
                       Conditions
                       -   Message : (Error msg in case of failure)
                           Reason : It is status of taskrun (Succeeded or Failed)
                       Podname (Pod name of the taskrun)
                       Steps
                       -   Container : (container name of the step in pod)
                           Name: (name of the step in task)
                           Terminated:
                           -   exitCode: (exit code of the container)
                               Reason: (status of container)
                       
                       (This will be repeated for all the steps in the task)
           (This will be repeated for all the tasks in the pipeline)
       ```
 - Tekton also helps analyze a failed PipelineRun or TaskRun.

    1. Look at the **status.condtions[0].message**

       In case of step failure, its shows **kubectl logs** command to fetch the logs of failed container 

    2. **Status.conditions[0].message** shows verbose error message e.g. validation failure 

### Is there a simpler and a better way?

The above process to see the logs and fetch the details seems very cumbersome, and slightly irritating too if it needs to be done very often. To simplify the above process, Tekton provides a [CLI](https://github.com/tektoncd/cli).

 - To get the list of all pipelineRuns, use:

   ```shell script
   tkn pipelinerun ls
   ```

 - To see the logs, run:

   ```shell script
   tkn pipelinerun logs <pipelinerun-name>
   ```

 - To see the details of the Pipelinerun, such as it’s status, Taskruns etc., run:

   ```shell script
   tkn pipelinerun describe <pipelinerun-name>
   ```

You can do the same for TaskRun too.

### Conclusion

I hope you find this post helpful and useful. In my next post, I will prepare an exercise to debug pipeline and demonstrate it. Stay tuned!


