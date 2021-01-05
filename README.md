# Iguazio Troubleshooting Guide

## Concepts
### Pods
What is a pod?
Why should I care?

### Jobs
What is a job?
How to submit job

## General Steps
### MLRun UI
Adi guide

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