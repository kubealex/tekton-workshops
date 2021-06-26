# Build and deploy your application with SpringBoot, Tekton and Kaniko

In this quick workshop, we are going to see how we can build and run our Java application in a Kubernetes cluster using a simple Tekton pipeline.

## What do we need?
- A running kubernetes cluster (minikube, kind, whatever!)
- A minimal persistentVolumeClaim
- Tekton installed in your cluster
- A quay.io or hub.docker.com account (for image pushing, a custom registry is ok too, reachable from the cluster)
- 15 min of your time

# Let's start!

## Clone the workshop repository
  
First of all, you need to clone this repository in your local folder!

    git clone https://github.com/kubealex/tekton-ws.git

Move to the folder of this workshop:

    cd springboot-build-deploy

You will find these files:

    .
    ├── README.md
    ├── pipeline.yml
    ├── pipelineRun.yml
    ├── resources
    │   ├── image-output.yml
    │   ├── k8s
    │   │   ├── pv.yml
    │   │   ├── roleBinding.yml
    │   │   └── serviceAccount.yml
    │   └── maven-settings-cm.yml
    └── tasks
        ├── git-clone.yml
        ├── k8s-deploy.yml
        ├── kaniko-build.yml
        └── maven.yml

We will go through each of them but some are self-explanatory.

The source code is a simple SpringBoot starter, **hello-springboot**, and its source code can be found here [https://github.com/kubealex/hello-springboot](https://github.com/kubealex/hello-springboot)

The manifests repository, a simple *service* and a *deployment* are available in a separated repository, here [https://github.com/kubealex/hello-manifests](https://github.com/kubealex/hello-manifests)

## Make a persistentVolume available  
  
The workshop requires a PV to be present, even a simple hostpath PV will fit the need! 

I put a simple PV inside the resources/k8s folder, in order to create a PV out of an HostPath in the machine.

This will act as a common workspace for all our tasks, to simplify the execution of the pipeline.

## Create a namespace for our resources  
  
Tekton will look for its resources **cluster-wide**, so we are not forced to use the *tekton-pipelines* namespace.

To dedicate a namespace to our activities, let's create one:

    kubectl create ns tekton-workshop

## Deploy the service account and the roleBinding for our tekton user  
  
Tekton pipelines allow the user to specify a serviceAccount to use while running in k8s.

In our case, we will be using the Service Account both for managing **Registry Credentials** and for binding it to the 'admin' role inside the namespace.

To proceed, let's deploy our resources, from the root folder of **springboot-build-deploy**:

    kubectl apply -f resources/k8s/serviceAccount.yml -n tekton-workshop
    kubectl apply -f resources/k8s/roleBinding.yml -n tekton-workshop

In the Service Account definition, I also bound a secret to it, called **dockercreds** that will contain our credentials for accessing our registry to push the image once built.

To create it you can just run the following command:

    kubectl create secret docker-registry dockercreds --docker-server=<YOUR_REGISTRY_URL> --docker-username=<YOUR_USERNAME> --docker-password=YOUR_PASSWORD --docker-email=<YOUR_EMAIL> -n tekton-workshop

With **YOUR_REGISTRY_URL** being the hostname of the docker registry service (like quay.io or hub.docker.com).

Check that the previous steps are correctly done, by typing:

    kubectl get sa,rolebinding,secrets -n tekton-workshop

    NAME                       SECRETS   AGE
    serviceaccount/tekton-sa   2         3h9m

    NAME                                                 ROLE                AGE
    rolebinding.rbac.authorization.k8s.io/tekton-admin   ClusterRole/admin   3h3m

    NAME                           TYPE                                  DATA   AGE
    secret/dockercreds             kubernetes.io/dockerconfigjson        1      4d23h
    secret/tekton-sa-token-fqlcq   kubernetes.io/service-account-token   3      3h9m

## Deploy Tekton resources

Now, we can start deploying all Tekton resources used in the example.

We have:
- PipelineResources - For defining common resources that will be used in the pipeline
- Tasks - The core of our pipeline, the definition of the tasks used in the pipeline
- Pipeline - The pipeline itself, where we specify the tasks to run, the resources they are going to use.
- PipelineRun - The *instance* of the pipeline to run. It references a pipeline to run, specifying runtime parameters like serviceAccount, volumes, resources.

**Deploy PipelineResource**

In the folder */resources* you can find the file *image-output.yml*, that defines a PipelineResource, kind *image* that will represent our output image:

    apiVersion: tekton.dev/v1alpha1
    kind: PipelineResource
    metadata:
    name: output-image-hello-world
    spec:
    type: image
    params:
        - name: url
        value: <REGISTRY_HOST>/<USERNAME>/<IMAGE_NAME>:<TAG> # ie: quay.io/kubealex/hello-tekton:latest

Edit the file before appyling it, pointing to your user 'space' in the registry, then:

    kubectl apply -f resources/image-output.yml -n tekton-workspace

**Deploy Tasks** 

In the *tasks* folder you will find all tasks:

- git-clone.yml
- maven.yml
- kaniko-build.yml
- k8s-deploy.yml

Each task is self consistent, and recalled in the pipeline with the correct parameters. Let's have a quick look to them.

*git-clone*

Along with Maven task, it is available in [Tekton Catalog](https://github.com/tektoncd/catalog).
It's a simple task that takes a repository URL as input, clones it in the workspace and makes the files available for further executions

*maven*

Available in Tekton Catalog too, this task has 2 steps inside it.
The first one, namely *maven-settings* prepares the settings.xml file if none is provided.
The second one, *mvn-goals* runs the 'mvn package' command in the directory where the source code has been cloned.

Since we will be using a standard settings.xml that we are going to provide via ConfigMap, let's define it!

    kubectl apply -f resources/maven-settings-cm.yml

*kaniko-build* 

This taks instantiates a Kaniko pod passing some arguments, like the location of the **Dockerfile**, the build context/folder, and the image output name.
This step is the reason why we defined the Service Account and the registry credentials, since it is going to build and push the image to the registry.

*k8s-deploy*

This task instantiates a custom image that I prepared with *kustomize* and *kubectl* ([Check it out here](https://github.com/kubealex/kubekustomize)) to edit and apply the manifests.


Proceed applying them:

    kubectl apply -f tasks/git-clone.yml -n tekton-workshop
    kubectl apply -f tasks/maven.yml -n tekton-workshop
    kubectl apply -f tasks/kaniko-build.yml -n tekton-workshop
    kubectl apply -f tasks/k8s-deploy.yml -n tekton-workshop


**Deploy Pipeline**

As a final step, we are going to deploy the Pipeline itself, that will orchestrate all tasks and correctly link the resources to our operations.

If you have a look at it, you will see that each step is mandatory for the next one, and each of them reference different parameters.

The pipeline will:

- Fetch the code from the source code repository
- Run the mvn package step on the freshly created workspace (our persistent volume!)
- Run kaniko build to build the container image using the provided dockerfile in the source code repo
- Fetch manifests from the manifest repository
- Run kustomization on the manifests and apply the manifests in the dedicated namespace.

## Run the pipeline!

Now that everything is in place, it's time for us to finally run the pipeline!

To do this, simply apply the PipelineRun by running:

    kubectl apply -f pipelineRun.yml -n tekton-workshop

This will automatically trigger the pipeline and you can check the logs by typing:

    tkn pipeline logs -f build-and-deploy -n tekton-workshop

Or you can also check a nice progress description with:

    tkn pipeline describe build-and-deploy -n tekton-workshop

If everything went good, you will find this situation:

    > tkn pipelinerun describe -n tekton-workshop
    Name:              build-and-deploy
    Namespace:         tekton-workshop
    Pipeline Ref:      build-and-deploy
    Service Account:   tekton-sa
    Timeout:           1h0m0s
    Labels:
    tekton.dev/pipeline=build-and-deploy

    🌡️  Status

    STARTED         DURATION    STATUS
    8 minutes ago   6 minutes   Succeeded

    📦 Resources

    NAME          RESOURCE REF
    ∙ app-image   output-image-hello-world

    ⚓ Params

    No params

    📝 Results

    No results

    📂 Workspaces

    NAME                 SUB PATH   WORKSPACE BINDING
    ∙ maven-settings     ---        ConfigMap (config=custom-maven-settings)
    ∙ shared-workspace   ---        VolumeClaimTemplate

    🗂  Taskruns

    NAME                                                  TASK NAME                    STARTED         DURATION     STATUS
    ∙ build-and-deploy-kustomize-manifests-wg24z          kustomize-manifests          2 minutes ago   9 seconds    Succeeded
    ∙ build-and-deploy-fetch-manifests-repository-cml8z   fetch-manifests-repository   2 minutes ago   7 seconds    Succeeded
    ∙ build-and-deploy-kaniko-build-nzt2f                 kaniko-build                 8 minutes ago   5 minutes    Succeeded
    ∙ build-and-deploy-maven-run-5v7r4                    maven-run                    8 minutes ago   27 seconds   Succeeded
    ∙ build-and-deploy-fetch-source-repository-r8znz      fetch-source-repository      8 minutes ago   5 seconds    Succeeded

    ⏭️  Skipped Tasks

    No Skipped Tasks


## Test your application!

Now that everything is in place, test your fantastic application running in K8S! 

    kubectl port-forward svc/hello-tekton 8080 -n tekton-workshop

And see that it is correctly running:

    > curl localhost:8080
    Hello Tekton!!

Congrats on running your pipeline successfully! 

## Cleanup
To cleanup your environment:

    kubectl delete Tasks,PipelineRun,Pipeline,PipelineResources,svc,deployment,cm,secret,sa -n tekton-workshop
    kubectl delete ns tekton-workshop

