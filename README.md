# Iguazio Troubleshooting Guide

## Table of Contents
- [Concepts](#Concepts)
    - [Pods](#Pods)
    - [Jobs](#Jobs)
    - [Pipelines](#Pipelines)
- [General Steps](#General-Steps)
    - [MLRun UI](#MLRun-UI)
    - [Nuclio UI](#Nuclio-UI)
    - [Grafana Dashboard](#Grafana-Dashboard)
    - [Jupyter Terminal](#Jupyter-Terminal)
- [Runtime Specific Steps](#Runtime-Specific-Steps)
    - [KubeJob (Job)](#KubeJob-Job)
    - [MPIJob (Horovod)](#MPIJob-Horovod)
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
View MLRun function code, artifacts, inputs, parameters, and more all per specific run. Organized by project.

### Nuclio UI
View Nuclio function configuration and logs in UI. Organized by project.

### KubeFlow Pipeline UI
Monitor pipeline execution and view pod logs.

### Grafana Dashboard
Monitor CPU/Mem/GPU utilization with pre-built or custom dashboards. Set up alerts in Slack or email when a particular value exceeds a specified limit.

### Jupyter Terminal
#### Kubectl
The main tool in the Jupyter Terminal is the `kubectl` command. We can use this to create, view, modify, and delete Kubernetes resources. It is an incredibly deep tool, but we only need a subset of the functionality.

The main pattern we will follow is:
```
kubectl <ACTION> <RESOURCE-TYPE> <POD-NAME>
or 
kubectl <ACTION> <RESOURCE-TYPE>
```

Examples include:
```
kubectl get pods
kubectl describe pod training-demo-c2z6f
kubectl delete pod training-demo-c2z6f
kubectl logs training-demo-c2z6f
```

We can combine `kubectl` with other command line tools such as `grep` or `awk` to filter results and find what we need.

#### Label Selection
Another useful way to use `kubectl` is called "label selection". Some pods will have values called `labels` that we can use to filter our results. It is essentially a KV filter for pods.

The main pattern we will follow is:
```
kubectl <ACTION> <RESOURCE-TYPE> -l <KEY>=<VALUE>
```

Examples include:
```
# MLRun
kubectl get pods -l mlrun/class=job
kubectl get pods -l mlrun/function=train-model
kubectl get pods -l mlrun/name=training-demo
kubectl get pods -l mlrun/owner=nick
kubectl get pods -l mlrun/project=sk-project

# Nuclio
kubectl get pods -l nuclio.io/project-name=getting-started-iris-marcelo
kubectl get pods -l nuclio.io/function-name=getting-started-iris-marcelo-model-server
kubectl get pods -l nuclio.io/class=function
kubectl get pods -l nuclio.io/function-version=latest

# MPIJob (Horovod)
kubectl get pod -l mpi-job-role=launcher
kubectl get pod -l mpi-job-role=worker

# App Services
kubectl get pods -l app=spark
kubectl get pods -l app=presto
kubectl get pods -l app=grafana
```

These label selectors can be found by running `kubectl describe pod <POD-NAME>` on any pod:
```
kubectl describe pod training-demo-c2z6f
...
Labels:             mlrun/class=job
                    mlrun/function=train-model
                    mlrun/name=training-demo
                    mlrun/owner=nick
                    mlrun/project=sk-project
                    mlrun/scrape-metrics=False
                    mlrun/tag=latest
                    mlrun/uid=39156d7275dc481a9b82b7fa9544e9ef
...
```

We will use a combination of `grep` and label selectors to find and filter pods when debugging.

## Runtime Specific Steps
To troubleshoot specific runtime jobs, you will need to use a combination of the MLRun UI, Grafana Dashboard, and `kubectl` commands via the Jupyter Terminal.

### KubeJob (Job)
#### List Jobs
```
kubectl get pods -l mlrun/class=job
NAME                  READY   STATUS      RESTARTS   AGE
download-wqq5v        0/1     Completed   0          13m
label-wtxzs           0/1     Completed   0          12m
training-demo-c2z6f   1/1     Running     0          43s
```

#### Describe Job in detail
```
kubectl describe pod training-demo-c2z6f
```

#### Get the logs or monitor job execution (to get execution details)
```
kubectl logs training-demo-c2z6f
```

#### Follow logs in real-time
```
kubectl logs training-demo-c2z6f -f
```

#### Stop the job
```
kubectl delete pod training-demo-c2z6f
```

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

#### Describe MPIJob in detail
```
kubectl describe mpijob train-75c6bb4a
```

#### Get the launcher pod for each job
```
kubectl get pods | grep train | grep launch
or
kubectl get pod -l mpi-job-role=launcher

train-75c6bb4a-launcher                                  1/1     Running            0          2d12h
train-e4fc59c5-launcher                                  1/1     Running            0          2d12h
```
The first method is simpler, but requires your `MPIJob` to be named "train". The second uses a label selector to filter the pods and is universal regardless of naming.

#### Get the logs or monitor job execution (to get execution details)
```
kubectl log train-75c6bb4a-launcher
```
You can see any logs from your job, such as the Epoch the job is at.

#### Follow logs in real-time
```
kubectl log train-75c6bb4a-launcher -f
```

#### Stop the MPIJob
```
kubectl delete mpijob train-75c6bb4a
```

#### Identify the MPIJob with GPU enabled
```
kubectl get mpijob train-e4fc59c5 -o yaml | grep -i gpu
```

### Nuclio
#### List Nuclio serverless functions
View functions within project in Nuclio UI or:
```
kubectl get pod -l nuclio.io/class=function
NAME                                                              READY   STATUS    RESTARTS   AGE
nuclio-default-recognize-faces-6747c89c8d-jj4ml                   1/1     Running   0          35d
nuclio-dogs-vs-cats-demo-deploy-model-5f8556cf7b-zxvt5            1/1     Running   0          21d
```

#### Describe Nuclio function in detail
View function in Nuclio UI or:
```
kubectl describe pod nuclio-default-recognize-faces-6747c89c8d-jj4ml
```

#### Get the logs or monitor execution
View logs in Nuclio UI or:
```
kubectl logs nuclio-default-recognize-faces-6747c89c8d-jj4ml
```

#### Follow logs in real-time
View logs in Nuclio UI or:
```
kubectl logs nuclio-default-recognize-faces-6747c89c8d-jj4ml -f
```

#### Stop the Nuclio function
Delete function in Nuclio UI or:
```
kubectl delete pod nuclio-default-recognize-faces-6747c89c8d-jj4ml
```

## FAQ
- What if I forget what the name of my pod is?
    - If executed via MLRun
        - Check MLRun UI or
        - `kubectl get pods -l mlrun/owner=<MY-NAME>`
    - If executed via Kubeflow Pipelines
        - Check KF Pipelines UI component log or 
        - `kubectl get pods -l mlrun/owner=<MY-NAME>`
    - If Nuclio serverless function
        - Check Nuclio UI or
        - `kubectl get pods -l nuclio.io/project-name=<MY-PROJECT>`
        - `kubectl get pods -l nuclio.io/function-name=<MY-FUNCTION>`
- How do I find if a job is still running?
    - `kubectl get pod <POD-NAME>` should give a status. If that status is not `Running`, the job is not running.
    - Check logs in UI or via `kubectl logs <POD-NAME>`. See above for specific runtime example.
- My job failed. Why?
    - Run `kubectl describe pod <POD-NAME>` on failure. For example `kubectl describe pod training-demo-c2z6f | grep -i state -C 5`. This will show the state and a brief description in case of failure.
    - For more inforamtion, check logs in UI or via `kubectl logs <POD-NAME>`. See above for specific runtime example.
- How can I find who launched a specfic MLRun pod?
    - Get the pod name
    ```
    kubectl get pods -l mlrun/class=job
    NAME                  READY   STATUS      RESTARTS   AGE
    download-wqq5v        0/1     Completed   0          53m
    label-wtxzs           0/1     Completed   0          52m
    training-demo-8qlcm   0/1     Completed   0          19m
    training-demo-c2z6f   0/1     Completed   0          40m
    ```
    - Describe pod and `grep` by `mlrun/owner`
    ```
    kubectl describe pod training-demo-c2z6f | grep mlrun/owner
    mlrun/owner=nick
    ```