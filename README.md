# Iguazio Troubleshooting Guide

## Table of Contents
- [Concepts](#Concepts)
    - [Pods](#Pods)
    - [Jobs](#Jobs)
    - [Pipelines](#Pipelines)
- [General Steps](#General-Steps)
    - [MLRun UI](#MLRun-UI)
    - [Grafana Dashboard](#Grafana-Dashboard)
    - [Jupyter Terminal](#Jupyter-Terminal)
- [Runtime Specific Steps](#Runtime-Specific-Steps)
    - [KubeJob (Job)](#KubeJob-(Job))
    - [MPIJob (Horovod)](#MPIJob-(Horovod))
    - [Nuclio](#Nuclio)
- [FAQ](#FAQ)

## Concepts
### Pods
#### What is a pod?
The shared context of a Pod is a set of Linux namespaces, cgroups, and potentially other facets of isolation - the same things that isolate a Docker container. Within a Pod's context, the individual applications may have further sub-isolations applied.

In terms of Docker concepts, a Pod is similar to a group of Docker containers with shared namespaces and shared filesystem volumes.

Source: [K8s Docs](https://kubernetes.io/docs/concepts/workloads/pods/#what-is-a-pod)

#### Why should I care about pods?
Whether we are working in the UI or deep in the command line, we are utilizing pods. While most of what Kubernetes is doing behind the scenes is not entirely relevant and can be abstracted away, pods are so fundamental that it helps to be aware of them and how we can find them for troubleshooting purposes (if necessary). This will be covered in detail later.

### Jobs
#### What is a job?
A Job creates one or more Pods and ensures that a specified number of them successfully terminate. As pods successfully complete, the Job tracks the successful completions. When a specified number of successful completions is reached, the task (ie, Job) is complete. Deleting a Job will clean up the Pods it created.

A simple case is to create one Job object in order to reliably run one Pod to completion. The Job object will start a new Pod if the first Pod fails or is deleted (for example due to a node hardware failure or a node reboot).

You can also use a Job to run multiple Pods in parallel.

Source: [K8s Docs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)

#### Why should I care about jobs?
MLRun jobs are the way that we will be launching our code. MLRun has support for many different runtimes such as `job`, `nuclio`, `mpijob`, `dask`, etc, but are all launched in the same manner.

#### How to submit a job?
The exact syntax to submit a job varies slighly by runtime. The general pattern is as follows:
1. Create MLRun project and define artifact path (place where any created/downloaded files will be deposited).
2. Define MLRun task that will be run when job is executed. This includes the code to run, specific runtime, and any parameters to run with. The code itself can be defined within the same notebook or in a separate script.
3. Submit MLRun task to be run locally (within notebook) or on cluster.

Examples can be found in the [examples](examples) directory.

### Pipelines 
#### What is a pipeline?
A pipeline is generally a series of components that do different tasks but are related in some way. There are many applications of a pipeline and many tools to create them. In Iguazio, we use [KubeFlow Pipelines](#https://www.kubeflow.org/docs/pipelines/).

For example, a sample machine learning pipeline might look like the following:
- Download Data
    - Downloads data from source such as S3
    - Saves raw data to MLRun databse
    - Passes raw data downstream
- Pre-Process Data
    - Performs data pre-processing or cleaning on raw data
    - Saves cleaned data to MLRun databse
    - Passes cleaned data downstream
- Train Model
    - Trains model with cleaned data
    - Saves training metrics and model to MLRun database
    - Passes trained model downstream
- Evaluate Model
    - Evaluates previously trained model
    - Saves testing metrics to MLRun database
- Deploy Model
    - Deploys previously trained model to REST API endpoint for testing or production

#### Why should I care about pipelines?
Standalone jobs are great, but when you have multiple jobs to run that are related or pass information between them, a pipeline is the way to go.

Another benefit is that each pipeline component can have its own code, runtime, container, etc. This allows for:
- Flexibility in design and implementation: Use the tools that are best for the job! Mix-and-match runtimes with ease.
- Resource management: Not every step of your pipeline needs to be run in a distrubted framework such as `Dask` or `Horovod` or a long-running `Nuclio` serverless function. Only deploy what you need.
- Separation between components: If a component errors out, you clearly know which pipeline component failed. Additionally, each pipeline component has its own separate logs for easy troubleshooting.


#### How to submit a KubeFlow pipeline?
1. Create MLRun project and define artifact path (place where any created/downloaded files will be deposited).
2. Define MLRun functions will be run when job is executed. This includes the code to run and the runtype kind. The code itself can be defined within the same notebook or in a separate script.
3. Add MLRun functions to MLRun project.
4. Define Kubeflow Pipeline in Python code. This includes parameters for each component as well as relationships between different components. For example, the input to one component can be the output from another.
5. Save Kubeflow Pipeline and submit to cluster.

Examples can be found in the [examples](examples) directory.

## General Steps
### MLRun UI
Adi guide

### KubeFlow Pipeline UI
Monitor pipeline execution and view pod logs

### Grafana Dashboard
Check CPU/Mem/GPU utilization

### Jupyter Terminal
kubectl get/describe pod/mpijob
Label selection

## Runtime Specific Steps
To troubleshoot specific runtime jobs, you will need to use a combination of the MLRun UI, Grafana Dashboard, and `kubectl` commands via the Jupyter Terminal.

### KubeJob (Job)
kubectl get pod

### MPIJob (Horovod)
`MPIJobs` will have two types of pods: launchers and workers.
- The launchers are the coordinators that the workers report back to. The launchers will also have all the log data.
- The workers are what receive the CPU/GPU/Memory resources and actually execute the code. If there are resource allocation issues, it is most likely the worker that is having problems (especially of GPU's are involved).

#### List MPIJobs (Horovod)
```
kubectl get mpijobs
NAME             AGE
train-75c6bb4a   2d12h
train-e4fc59c5   2d12h
```

#### Get the launcher pod for each job
```
kubectl get pods | grep train | grep launch
train-75c6bb4a-launcher                                  1/1     Running            0          2d12h
train-e4fc59c5-launcher                                  1/1     Running            0          2d12h
```

#### Get the logs or monitor job execution (to get execution details)
```
kubectl log train-75c6bb4a-launcher
```
You can see any logs from your job, such as the Epoch the job is at.

#### Stop the MPIJob
```
kubectl delete mpijob train-75c6bb4a
```

#### Identify the MPIJob with GPU enabled
```
kubectl get mpijob train-e4fc59c5 -o yaml | grep -i gpu
```

### Nuclio
kubectl get pod -l nuclio.io/class=function

## FAQ
- What if I forget what the name of my pod is?
- How do I find if a job is still running?
- My job failed. Why?
- How can I find who launched a specfic MLRun pod?