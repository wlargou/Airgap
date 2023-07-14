# Prerequisites :
- Kubernetes cluster (Airgap - No internet)
- Private Image Registry (In this tutorial, we are using a Docker registry running on our kubernetes)
- Bastion host with internet connectivity
- Nerdctl installed

# Procedure : 
Tekton installation in connected mode is pretty straighforward, but it can be pretty challenging in airgapped environment, the first step is to pull required images.

## Step 1 : Image identification 
To identify the required images for Tekton deployment, download the latest release.yaml from Tekton :
[Tekton Official Website](https://tekton.dev/docs/installation/pipelines/)

`wget https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml`

In the release yaml the images are mentionned, there are two types of images, ones required for the installation and ones required for running the pipelines 

> Hint : Look for "image:" and "args" for the tekton controller

At the time of writing we identified the following images :

1. "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.49.0@sha256:968999c9f4ba1003725a9455f9a3a2cba36766768e4f1ee40010fafa765f450d"
2. "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/resolvers:v0.49.0@sha256:78ddd51c8dda6e1e8aa0d3ee65f49f76c9f7bde8235320ae81db2d9ed0e6ce32"
3. "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.49.0@sha256:df3be59025cc59dbcc639710a77f922f07b778de49616b59bb5343fbf7cc8b79"
4. "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.49.0@sha256:cde0654aab99ea19e030eb269f28deba6cc550910586ee7a832cae3ee63ea565"
5. "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/entrypoint:v0.49.0@sha256:0e43b6ae2d517df85aac356b411fe291057c2f12aef3a949be961cfc1d31c158" 
6. "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/nop:v0.49.0@sha256:91eb79439e756e557259da3c0823f29483863ed6b8a409da664f879279c95d59" 
7. "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/sidecarlogresults:v0.49.0@sha256:4055c213dbb60722432c87b80fb8e52ed6409e6cbb83e62ebb53f0c6d33056f6" 
8. "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/workingdirinit:v0.49.0@sha256:643cf8dbc46fbbfb9f333628c33bbdfb76d11b5005c2aaed28abdc20f739d0b8"
9. "cgr.dev/chainguard/busybox@sha256:19f02276bf8dbdd62f069b922f10c65262cc34b710eea26ff928129a736be791"
10. "mcr.microsoft.com/powershell:nanoserver@sha256:b6d5ff841b78bdf2dfed7550000fd4f3437385b8fa686ec0f010be24777654d6"
11. "docker.io/library/alpine:latest" - This one will be used to test

> Note : Image pull from microsoft will fail since we are running on linux

## Step 2 : Pull images in the bastion host 
- Next step is to pull the images on the bastion with internet connectivity :

`nerdctl pull "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/entrypoint:v0.49.0@sha256:0e43b6ae2d517df85aac356b411fe291057c2f12aef3a949be961cfc1d31c158"`

## Step 3 : Save images to disk
- Create a folder to hold all the compressed images and run for all images :

`nerdctl save -o tekton-events.tar "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.49.0@sha256:968999c9f4ba1003725a9455f9a3a2cba36766768e4f1ee40010fafa765f450d"`

- Then copy all the compressed images to one of your Kubernetes nodes (scp, usb, ...)

## Step 4 : Preparing images 
- Login to your registry :

`registry_url=$(kubectl get svc registry -n kurl -o=jsonpath='{.spec.clusterIP}'):443`
`registry_user=$(kubectl get secret registry-creds -n default -o=jsonpath='{.data.\.dockerconfigjson}' | base64 --decode | grep -o '"username":"[^"]\+"' | cut -d':' -f2 | tr -d '"')`
`registry_pass=$(kubectl get secret registry-creds -n default -o=jsonpath='{.data.\.dockerconfigjson}' | base64 --decode | grep -o '"password":"[^"]\+"' | cut -d':' -f2 | tr -d '"')`

`nerdctl login -u $registry_user -p $registry_pass $registry_url`

- Then load all your images :

`nerdctl -n k8s.io load -i tekton-events.tar`


- Verify all images are loaded :

`nerdctl -n k8s.io image ls` 

- Tag your images :

`nerdctl -n k8s.io image tag gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.49.0@sha256:968999c9f4ba1003725a9455f9a3a2cba36766768e4f1ee40010fafa765f450d $registry_url/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.49.0`


- Finally push the images to the registry :

`nerdctl -n k8s.io push $registry_url/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.49.0`


## Step 5 : Start the deployment 
Open release.yaml and edit all the references pointing to the online registry to reference your local registry : my_private_registry:443/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.49.0.

- Finally run the deployment :

`kubectl apply -f release.yaml`

- And check the status of the pods :

`kubectl get pods -n tekton-pipelines`

# Tekton Test

To test our deployment, let's create a simple task :

- Create a file for the task hello-world.yaml : 
```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: echo
      image: my_private_registry/library/alpine:latest
      script: |
        #!/bin/sh
        echo "Hello World"
```

- Then create a file for the task run hello-world-run.yaml
```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: hello-task-run
spec:
  taskRef:
    name: hello
```

- Apply the files created :

`kubectl apply --f hello-world.yaml`
`kubectl apply --f hello-world-run.yaml`

- Check results :

`kubectl get taskrun hello-task-run`

**Example :** 
NAME                    SUCCEEDED    REASON       STARTTIME   COMPLETIONTIME
 hello-task-run          True         Succeeded    22h         22h

- Check output of the task 

`kubectl logs --selector=tekton.dev/taskRun=hello-task-run`

# Tested versions :
