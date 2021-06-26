# Build your SpringBoot application and containerize it with Tekton and Kaniko

In this quick workshop, we are going to see how we can build our Java application in a Kubernetes cluster using a simple Tekton pipeline.

## What do we need?
- A running kubernetes cluster (minikube, kind, whatever!)
- Tekton installed in your cluster
- A quay.io or hub.docker.com account (for image pushing, a custom registry is ok too, reachable from the cluster)
- 10 min of your time

# Let's start!

## Clone the workshop repository
  
First of all, you need to clone this repository in your local folder!

    git clone https://github.com/kubealex/tekton-ws.git

Move to the folder of this workshop:

    cd kaniko-image-build

You will find these files:

    .
    ├── README.md
    ├── registry-creds.yml
    ├── resources
    │   ├── git-repo.yml
    │   ├── image-output.yml
    │   └── k8s
    │       └── serviceAccount.yml
    ├── task-run.yml
    └── tasks
        └── run-kaniko-build.yml

We will go through each of them but some are self-explanatory.

The source code is a simple SpringBoot starter, **hello-springboot**, and its source code can be found here [https://github.com/kubealex/hello-springboot](https://github.com/kubealex/hello-springboot)

## Create a namespace for our resources  
  
Tekton will look for its resources **cluster-wide**, so we are not forced to use the *tekton-pipelines* namespace.

To dedicate a namespace to our activities, let's create one:

    kubectl create ns tekton-workshop

## Deploy the service account and the registry credentials for our tekton user  
  
Tekton pipelines allow the user to specify a serviceAccount to use while running in k8s.

In our case, we will be using the Service Account both for managing **Registry Credentials**.

To proceed, let's deploy our resources, from the root folder of **kaniko-image-build**:

    kubectl apply -f resources/k8s/serviceAccount.yml -n tekton-workshop

In the Service Account definition, I also bound a secret to it, called **dockercreds** that will contain our credentials for accessing our registry to push the image once built.

To create it you can just run the following command:

    kubectl create secret docker-registry dockercreds --docker-server=<YOUR_REGISTRY_URL> --docker-username=<YOUR_USERNAME> --docker-password=YOUR_PASSWORD --docker-email=<YOUR_EMAIL> -n tekton-workshop

With **YOUR_REGISTRY_URL** being the hostname of the docker registry service (like quay.io or hub.docker.com).

Check that the previous steps are correctly done, by typing:

    kubectl get sa,secrets -n tekton-workshop

    NAME                       SECRETS   AGE
    serviceaccount/tekton-sa   2         3h9m

    NAME                           TYPE                                  DATA   AGE
    secret/dockercreds             kubernetes.io/dockerconfigjson        1      4d23h
    secret/tekton-sa-token-fqlcq   kubernetes.io/service-account-token   3      3h9m

## Deploy Tekton resources

Now, we can start deploying all Tekton resources used in the example.

We have:
- Tasks - The core of our execution, the definition of the task that we will run
- TaskRun - The trigger to execute our task
- Resources - The inputs and outputs of our task!

**Deploy input and output resources**

In the *resources* folder, there are these two PipelineResources, one for a *git repository* and one for the *built image*:

- git-repo.yml

It's a simple resource that identifies a git repository, that is automatically fetched and cloned during the execution, becoming the actual workspace for the task

- image-output.yml

It's another simple resource that identifies a container image URL, and it will be used during the *push* phase.

Let's deploy them!


    kubectl apply -f git-repo.yml -n tekton-workshop

    kubectl apply -f image-output.yml -n tekton-workshop


**Deploy Tasks** 

In the *tasks* folder you will find the core task:

- kaniko-build.yml

The task is self consistent, and uses an input (the **git-repo** resource) and an input (the **builtImage** resource)

*kaniko-build* 

This taks instantiates a Kaniko pod passing some arguments, like the location of the **Dockerfile**, the build context/folder, and the image output name.
This step is the reason why we defined the Service Account and the registry credentials, since it is going to build and push the image to the registry.

    kubectl apply -f kaniko-build.yml -n tekton-workshop

## Run the task!

Now that everything is in place, it's time for us to finally run the task!

To do this, simply apply the TaskRun by running:

    kubectl apply -f taskRun.yml -n tekton-workshop

This will automatically trigger the pipeline and you can check the logs by typing:

    tkn pipeline logs -f run-kaniko-build -n tekton-workshop

Or you can also check a nice progress description with:

    tkn taskrun describe -n tekton-ws

If everything went good, you will find this situation:


    Name:              run-kaniko-build
    Namespace:         tekton-workshop
    Task Ref:          build-kaniko-image
    Service Account:   kaniko-sa
    Timeout:           1h0m0s
    Labels:
    app.kubernetes.io/managed-by=tekton-pipelines
    tekton.dev/task=build-kaniko-image

    🌡️  Status

    STARTED          DURATION    STATUS
    34 minutes ago   6 minutes   Succeeded

    📨 Input Resources

    NAME         RESOURCE REF
    ∙ git-repo   hello-springboot-git

    📡 Output Resources

    NAME           RESOURCE REF
    ∙ builtImage   output-image-hello-world

    ⚓ Params

    NAME                 VALUE
    ∙ pathToDockerFile   Dockerfile.build

    📝 Results

    No results

    📂 Workspaces

    No workspaces

    🦶 Steps

    NAME                            STATUS
    ∙ create-dir-builtimage-7g2v4   Completed
    ∙ git-source-git-repo-vzqch     Completed
    ∙ build-and-push                Completed
    ∙ image-digest-exporter-f5vns   Completed

    🚗 Sidecars

    No sidecars



## Test your application!

Now that everything is in place, test your fantastic application running as a container!

**Docker**

    docker run -it -p 8080:8080 <YOUR_IMAGE_URL>

**Podman**

    podman run -it -p 8080:8080 <YOUR_IMAGE_URL>

And see that it is correctly running:

    > curl localhost:8080
    Hello Tekton!!

Congrats on running your pipeline successfully! 

## Cleanup
To cleanup your environment:

    kubectl delete Tasks,TaskRun,PipelineResources,secret,sa -n tekton-workshop
    kubectl delete ns tekton-workshop

