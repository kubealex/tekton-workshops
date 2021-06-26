# Tekton Workshops

This repository has a simple goal: trying to simplify life of those who are moving the first steps into Tekton, providing some simple use cases that can be adapted and improved to fit your needs!

## What do we need to enjoy the workshops?
- A running kubernetes cluster (minikube, kind, whatever!)
- A minimal persistentVolume (hostPath is fine!)
- A quay.io or hub.docker.com account (for image pushing, a custom registry is ok too, reachable from the cluster)
- 15 min of your time each!

# Let's start!

### Clone the workshop repository

First of all, you need to clone this repository in your local folder!

    git clone https://github.com/kubealex/tekton-ws.git

You will find a simple workshop to play with:  

- [Build and deploy a SpringBoot application in your cluster](https://github.com/kubealex/tekton-ws/tree/main/workshops/springboot-build-deploy)
- [Build your SpringBoot application and containerize it with Tekton and Kaniko](https://github.com/kubealex/tekton-workshops/tree/main/workshops/kaniko-image-build)


As you can see in the content, each workshop contains its resources that are needed to run it.

    .
    ├── LICENSE
    ├── README.md
    └── workshops
        ├── kaniko-image-build
        │   ├── README.md
        │   ├── registry-creds.yml
        │   ├── resources
        │   │   ├── git-repo.yml
        │   │   ├── image-output.yml
        │   │   └── k8s
        │   │       └── serviceAccount.yml
        │   ├── taskRun.yml
        │   └── tasks
        │       └── kaniko-build.yml
        └── springboot-build-deploy
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

### Setup Kubernetes

At this point you should have a working k8s cluster up&running.

If you need further guidance on this point you take a look at the different setup choices:

- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)  - A k8s in Docker setup  
- [minikube](https://minikube.sigs.k8s.io/docs/) - A minimal single-VM install  
- [libvirt-provisioner](https://github.com/kubealex/libvirt-k8s-provisioner) - A full cluster installer on libvirt/KVM  

Once your cluster is ready, it's time to start installing our components!

### Installing tekton

You can follow instructions on Tekton's [website](https://tekton.dev/docs/getting-started/) for a full featured set of parameters, but for our needs we can simply go straight to the point :)

    kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

This will also create the needed 'tekton-pipelines' namespace for all components to be spawned.

    > kubectl get pod -n tekton-pipelines
    NAME                                           READY   STATUS    RESTARTS   AGE
    tekton-pipelines-controller-5f88bb8695-s9jpc   1/1     Running   0          13d
    tekton-pipelines-webhook-77d48dc65c-6l4qx      1/1     Running   0          13d


### Install Tekton CLI

Tekton has a good CLI to interact with resources and pretty-print outputs of their executions.

If you are on a Mac, use brew to install it:

    brew install tektoncd-cli

Otherwise, follow [the instructions](https://github.com/tektoncd/cli) to set it up on your platform.
