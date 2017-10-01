# Java One Hands on Labs: Build, Test, Deploy, Operate.

Build, test, deploy, operate; a 4 step tutorial to use Docker, K8S and Java in the real world Abstract:Come join the Wercker+Oracle team and learn how to build and deploy a Java application in4 easy, entirely automated steps using Docker, K8S and Java. This session is BYOL (Bring yourown laptop)

## Pre-requisites:

The Hands on Lab will be run take place inside of a Virtual Machine on a laptop provided by you. Your laptop must have the following installed to take part:

* Virtualbox
* Vagrant

You should have familiarity with the above and some understanding of the following will be beneficial to get the most out of the session, although is not a hard requirement:

* Linux
* Bash
* Docker
* Kubernetes

* Presentation
    * Explain Wercker
    * Explain CLI
    * Explain Virtual Machine method for the demo
    * Explain the application we're going to deploy

## Tutorial

### Develop

1. Fork the github application in to your own account
2. Clone the repo to your local laptop
3. Get VM running with `vagrant up`
4. `vagrant ssh` to get inside of the vm, then `sudo su -` to become root, followed by `cd /vagrant` to get to our working directory
5. Note that we don't have a Java runtime, nor Gradle installed on the VM. All we have is Docker.
6. run `wercker build`
7. This will fail. We need to edit source code.
8. We want to be able to test the source changes we make locally, so lets add a dev pipeline:

```
# Developer.
dev:
  steps:
    - script:
      name: gradle bootRun
      code: |
        ./gradlew bootRun
```

9. Fix the source by changing `return null;` to `return this.time;` in `src/main/java/time/Time.java`
10. Run `wercker dev --publish 8080`. This will come up on your development laptop on 8081 thanks to port forwarding config in the VM.
11. Test that this has come up by hitting `http://localhost:8081/time` on your laptop. At this point we've got an instance of our application built and run by the Wercker CLI inside of our VM.
11. Run `wercker build` to make sure our unit tests now pass. If they do Gradle will build a jar for the application.

### Build & Test

At this point we're happy that our code is good and our application should be able to build in any environment using the Wercker CLI, which will spin up a Docker container with all of the necesary dependencies and tooling, and end with a JAR artifact.

The next step is to create a self contained Docker image that just contains the JAR, configured a set of instructions for running our application wherever we want. We want to be able to create this Docker image using Wercker, and we want it to automatically run whenever we make a change to our application.

The process for this is:

14. Create a push-image pipeline:

```
# Push Docker Image
push-image:
    steps:
        - internal/docker-push:
            cmd: java -jar /pipeline/source/time-api.jar
            tag: $WERCKER_GIT_BRANCH
            ports: "8080"
            username: $DOCKER_USERNAME
            password: $DOCKER_PASSWORD
            repository: $DOCKER_REPO
```

15. Sign up to Wercker
18. Add the Wercker application
16. Sign up for Dockerhub: https://hub.docker.com/
17. Create a new private repo
19. Add the environment variables needed for Dockerhub:
    DOCKER_USERNAME
    DOCKER_PASSWORD
    DOCKER_REPO
20. Join the build and push docker pipelines in to a Workflow
21. Trigger your first Wercker Pipeline run by clicking "deploy now"
22. We now have an image.
23. Log in to Dockerhub registry from the VM with `docker login`.
24. Pull the image locally and do a `docker run -p 8080:8081` against it to show that it's come up and accessable

### Deploy & Operate

Now it's time to deploy it to Kubernetes. There are two ways of doing this. The _real world_ approach would be to configure everything we need to run on Wercker Web in our Pipelines project, and utilise environment variables defined in previous steps to automate the process of deploying. The demo land and development requirements are such that we want to deploy to a local instance of Kubernetes running inside of VMs. Since Wercker Web has no way of accessing our local machines without some networking craziness, we can utilise the Wercker CLI to run the Kubernetes deployment pipeline locally, with the simple caveat of needing to define some extra environment variables that would have otherwise been inferred online.

25. on your local machine, in the cloned repo, go to the `kubernetes-vagrant-coreos-cluster` directory, run `vagrant up`
26. Wait for Kubernetes to come up.
27. Linux/Mac users: `kubectl get nodes`, windows users, `vagrant ssh` then `kubectl get nodes`
28.. Look at the Kubernetes Service and Deployment files, notice the environment variables that need to be defined:
    a. IMAGEPULL_SECRET
    b. DOCKER_REPO
    c. WERCKER_GIT_BRANCH
29. Create a imagepull secret in Kubernetes `create secret docker-registry wercker-demo --docker-server=https://quay.io--docker-username=<QUAY USERNAME> --docker-password=<QUAY PASSWORD> --docker-email=<QUAY EMAIL>`
30. Write the deploy-to-kubernetes step in wercker.yml

```
# Deploy to a Kubernetes cluster
deploy-to-kubernetes:
    # We only need a minimal shell environment to run Kubectl.
    box:
        id: alpine
        cmd: /bin/sh
    steps:
    - script:
        name: Prepare Kubernetes files
        code: |
          ls -lth
          cat kubernetes_*.yml
          mkdir $WERCKER_OUTPUT_DIR/kubernetes
          mv kubernetes_*.yml $WERCKER_OUTPUT_DIR/kubernetes

    - kubectl:
        name: deploy to kubernetes
        server: $KUBERNETES_MASTER
        token: $KUBERNETES_TOKEN
        insecure-skip-tls-verify: true
        command: apply -f $WERCKER_OUTPUT_DIR/kubernetes/

    - kubectl:
        name: set deployment timeout
        server: $KUBERNETES_MASTER
        token: $KUBERNETES_TOKEN
        insecure-skip-tls-verify: true
        command: patch deployment/time-api -p '{"spec":{"progressDeadlineSeconds":60}}'

    - kubectl:
        name: check deployment status
        server: $KUBERNETES_MASTER
        token: $KUBERNETES_TOKEN
        insecure-skip-tls-verify: true
        command: rollout status deployment/time-api

```

31. Notice that we're defining more environment variables that will need to be replaced:
    a. KUBERNETES_MASTER
    b. KUBERNETES_TOKEN
32. Get the Kubernetes API IP address and token via `kubectl describe secret default-token-fb3f9`.
33. Create a file to hold the environment variables inside the VM at `local.env`:

```
IMAGEPULL_SECRET="wercker-demo"
DOCKER_REPO="< SAME DOCKER_REPO THAT'S ON WERCKER WEB >"
WERCKER_GIT_BRANCH="master"
KUBERNETES_MASTER="https://172.17.8.101 [IP ADDRESS THAT THE K8S MASTER VM SHOULD BE ON]"
KUBERNETES_TOKEN="< FROM THE KUBECTL COMMAND OUTPUT >"
```

34. `wercker build --pipeline=deploy-to-kubernetes --environment local.env`
35. Watch the output of the "check deployment status" step which will be polling Kubernetes as it attempts to start the time-api container.
36. GET IP ADDRESS OF SERVICE
37. ATTEMPT TO CURL API
