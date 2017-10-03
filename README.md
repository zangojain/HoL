# Java One Hands on Labs: Build, Test, Deploy, Operate.

Build, test, deploy, operate; a 4 step tutorial to use Docker, K8S and Java in the real world Abstract:Come join the Wercker+Oracle team and learn how to build and deploy a Java application in4 easy, entirely automated steps using Docker, K8S and Java. This session is BYOL (Bring yourown laptop)

## Pre-requisites:

The Hands on Lab will be run take place inside of a Virtual Machine on a laptop provided by you. Your laptop must have the following installed to take part:

* Virtualbox
* Vagrant

You should have familiarity with the above and a basic understanding of the following will be beneficial, although is not a hard requirement:

* Linux
* Bash
* Docker
* Kubernetes

## Tutorial

0. Consider running `vagrant up` in "kubernetes-vagrant-coreos-cluster" before doing anything else as it may take 15 mins to get the cluster running.

### Develop

1. Fork the github application in to your own account
2. Clone the repo to your local laptop
3. Get HoL VM running with `vagrant up`
4. `vagrant ssh` to get inside of the vm, then `sudo su -` to become root, followed by `cd /vagrant` to get to our working directory
5. Note that we don't have a Java runtime, nor Gradle installed on the VM. All we have is Docker.
6. run `wercker build`
7. This will fail. We need to edit source code.
8. We want to be able to test the source changes we make locally, so lets add a dev pipeline:

```
dev:
  steps:
    - script:
      name: gradle bootRun
      code: |
        ./gradlew bootRun
```

9. Fix the source by changing `return null;` to `return this.time;` in `src/main/java/time/Time.java`
10. Run `wercker dev --publish 8080`. This will come up on your development laptop on 8081 thanks to port forwarding config in the HoL VM.
11. Test that this has come up by hitting `http://localhost:8081/time` on your laptop. At this point we've got an instance of our application built and run by the Wercker CLI inside of our HoL VM.
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
23. Log in to Dockerhub registry from the HoL VM with `docker login`.
24. Pull the image locally and do a `docker run -p 8080:8080` against it to show that it's come up and accessable

### Deploy & Operate

Now it's time to deploy it to Kubernetes. There are two ways of doing this. The _real world_ approach would be to configure everything we need to run on Wercker Web in our Pipelines project, and utilise environment variables defined in previous steps to automate the process of deploying. The demo land and development requirements are such that we want to deploy to a local instance of Kubernetes running inside of VMs. Since Wercker Web has no way of accessing our local machines without some networking craziness, we can utilise the Wercker CLI to run the Kubernetes deployment pipeline locally, with the simple caveat of needing to define some extra environment variables that would have otherwise been inferred online.

25. on your local machine, in the cloned repo, go to the `kubernetes-vagrant-coreos-cluster` directory, run `vagrant up` if you haven't already. This may take some time.
27. Linux/Mac users: `kubectl get nodes`, windows users, `vagrant ssh` then `kubectl get nodes`
28.. Look at the Kubernetes Service and Deployment files, notice the environment variables that need to be defined:
    a. IMAGEPULL_SECRET
    b. DOCKER_REPO
    c. WERCKER_GIT_BRANCH
29. Create a imagepull secret in Kubernetes `kubectl create secret docker-registry wercker-demo --docker-username=< DOCKERHUB USERNAME > --docker-password="< DOCKERHUB PASSWORD >" --docker-email=< DOCKERHUB EMAIL >`
30. Write the deploy-to-kubernetes step in wercker.yml
31. First, start by preparing what we're going to send to Kubernetes, and define how we're going to authenticate:

```
deploy-to-kubernetes:
    box:
        id: alpine
        cmd: /bin/sh
    steps:

    - bash-template

    - script:
        name: Prepare Kubernetes files
        code: |
          ls -lth
          cat kubernetes_*.yml
          mkdir $WERCKER_OUTPUT_DIR/kubernetes
          mv kubernetes_*.yml $WERCKER_OUTPUT_DIR/kubernetes

    - create-file:
        name: Create Kubernetes CA
        filename: ca.crt
        overwrite: true
        content: $KUBERNETES_CA

    - create-file:
        name: Create Kubernetes Client Cert
        filename: cert.pem
        overwrite: true
        content: $KUBERNETES_CLIENT_CERT

    - create-file:
        name: Create Kubernetes Client Key
        filename: key.pem
        overwrite: true
        content: $KUBERNETES_CLIENT_KEY
```

32. Next, define the actual Kubectl interaction:

```
    - kubectl:
        name: deploy to kubernetes
        server: $KUBERNETES_MASTER
        certificate-authority: ca.crt
        client-certificate: cert.pem
        client-key: key.pem
        command: apply -f $WERCKER_OUTPUT_DIR/kubernetes/

```

31. Notice that we're defining more environment variables that will need to be replaced:
    a. KUBERNETES_MASTER
    b. KUBERNETES_CA
    c. KUBERNETES_CLIENT_CERT
    d. KUBERNETES_CLIENT_KEY

32. Our local Kubernetes cluster exposes files that contain our desired values for the certificates abovebe found via `kubectl config view`, listed under "client-certificate", "client-key", and "certificate-authority" for "default-cluster". You could use any of the other Kubernetes authentication methods here, but since our Vagrant cluster exposes certificates that we need to pass to the Wercker CLI as environment variables. This requires an additional step of getting getting the multi-line certificates as a string:

33. `awk '$1=$1' ORS='\\n' < FILE NAME >`

32. Create a file inside of the HoL VM to hold the environment variables inside the VM at `local.env`:

```
X_IMAGEPULL_SECRET="wercker-demo"
X_DOCKER_REPO="< SAME DOCKER_REPO THAT'S ON WERCKER WEB >"
X_WERCKER_GIT_BRANCH="master"
X_KUBERNETES_MASTER="https://172.17.8.101
X_KUBERNETES_CA=" < SINGLE LINE OUTPUT FROM AWK >"
X_KUBERNETES_CLIENT_CERT="< SINGLE LINE OUTPUT FROM AWK >"
X_KUBERNETES_CLIENT_KEY="< SINGLE LINE OUTPUT FROM AWK >"
```

34. `wercker --environment local.env build --pipeline deploy-to-kubernetes`
35. Jump back in to your terminal window where you launched the Kubernetes VM and run `kubectl get pods` to see if your pods came up successfully.

Congrats, you just built your application,created a docker image from the resulting artifact and deployed it to a local Kubernetes cluster in an entirely repeatable and independant way using the Wercker CLI.

However, the `kubectl apply` command's success simply shows whether the Kubernetes accepted the instructions we passed to it, it's not telling you whether or not they were successful. This means you had to run `kubectl get pods` outside of your build and deployment flow to find out whether it was successful, which is not ideal since it means we can't fully automate the end-to-end flow.

The good news is that we can add an additional few calls to our cluster from inside of Wercker:

```
    - kubectl:
        name: set deployment timeout
        certificate-authority: ca.crt
        client-certificate: cert.pem
        client-key: key.pem
        command: patch deployment/time-api -p '{"spec":{"progressDeadlineSeconds":240}}'

    - kubectl:
        name: check deployment status
        certificate-authority: ca.crt
        client-certificate: cert.pem
        client-key: key.pem
        command: rollout status deployment/time-api

```

36. Re-run your Kubernetes deployment pipeline in the Wercker CLI with `wercker --environment local.env build --pipeline deploy-to-kubernetes`. The output of the check deployment status step will wait up to 240 seconds for your new Kubernetes pod to become available. If the timeout is hit or the pods fail, the step will fail.


