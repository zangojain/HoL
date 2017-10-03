# Java One Hands on Labs: Build, Test, Deploy, Operate.

This repository includes everything that is needed to follow along with the Java One 2017 San Francisco Hands on Lab that I (@riceo) am running on Oct 3rd 2017. If you aren't reading this in the session, you should still be able to follow along!

This Hands on Lab will take a simple Java SprintBoot application that servers the current time over HTTP in JSON, ensure its unit tests pass, then create an artifact in the form of a JAR that we will then create and deploy a Docker image from.

We'll do all of this in a repeatable, automated way using Wercker. Since this is a Hands on Lab that you should be able to follow along with, I want to reduce the number of external dependencies, so we'll be deploying using the Wercker CLI and a local Kubernetes cluster.

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

### Develop

1. Fork the github application in to your own account
2. Clone the repo to your local laptop
3. Get VM running with `vagrant up`
4. In a separate terminal window, run `vagrant up` in `kubernetes-vagrant-coreos-cluster`, which will begin the provisioning of your local Kubernetes cluster. You may need to enter your admin password a few times over the next 15 minutes.
5. Back in the root git repo, run `vagrant ssh` to get inside of the vm, then `sudo su -` to become root, followed by `cd /vagrant` to get to our working directory
6. Note that we don't have a Java runtime, nor Gradle installed on the VM. All we have is Docker.
7. run `wercker build` to see if the existing wercker.yml we have will build our application.
8. It won't. Our tests are failing and we need to edit source code to fix it.
9. We want to be able to test the source changes we make locally, so lets add a dev pipeline:

```
dev:
  steps:
    - script:
      name: gradle bootRun
      code: |
        ./gradlew bootRun
```

10. Fix the source by changing `return null;` to `return this.time;` in `src/main/java/time/Time.java`
11. Run `wercker dev --publish 8080`. This will come up on your development laptop on 8081 thanks to port forwarding config in the VM.
12. Test that this has come up by hitting `http://localhost:8081/time` on your laptop.

At this point we've got an instance of our application built and run by the Wercker CLI inside of our VM.

13. Run `wercker build` to make sure our unit tests now pass. If they do Gradle will build a jar for the application.

### Build & Test

At this point we're happy that our code is good and our application should be able to build in any environment using the Wercker CLI, which will spin up a Docker container with all of the necessary dependencies and tooling, and end with a JAR artifact.

The next step in the process is to create a self contained Docker Image that just contains the JAR, configured to run our application wherever we want via a simple `docker run`. We want to be able to create this Docker image using Wercker, and we want it to automatically run whenever we make a change to our application.

The process for this is:

14. Create a push-image pipeline in wercker.yml:

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
16. Add the Wercker application
17. Sign up for Dockerhub: https://hub.docker.com/
18. Create a new private repo
19. Add the environment variables needed for Dockerhub:
    a. DOCKER_USERNAME
    b. DOCKER_PASSWORD
    c. DOCKER_REPO
20. Join the build and push docker pipelines in to a Workflow
21. Trigger your first Wercker Pipeline run by clicking "deploy now"
22. We now have an image.
23. Log in to Dockerhub registry from the VM with `docker login`.
24. Pull the image locally and do a `docker run -p 8080:8080` against it to show that it's come up and accessable

### Deploy & Operate

Now it's time to deploy our application to Kubernetes.

There are two ways of doing this. The _real world_ approach would be to configure everything we need to run on Wercker Web in our Pipelines project, and utilise environment variables defined in previous steps to automate the process of deploying to an external, internet-facing cluster.

Demo-land and development requirements are such that we want to deploy to a local instance of Kubernetes running inside of VMs. Since Wercker Web has no way of accessing our local machines without some networking craziness, we can utilise the Wercker CLI to run the Kubernetes deployment pipeline locally, with the caveat of needing to define some extra environment variables manually, that would have otherwise been inferred on Wercker Web.

25. on your local machine, in the cloned repo, go to the `kubernetes-vagrant-coreos-cluster` directory, run `vagrant up` if you haven't already. This may take some time.
26. Linux/Mac users: `kubectl get nodes`, windows users, `vagrant ssh` then `kubectl get nodes`
27.. Look at the Kubernetes Service and Deployment files, notice the environment variables that need to be defined:
    a. IMAGEPULL_SECRET
    b. DOCKER_REPO
    c. WERCKER_GIT_BRANCH
28. Create a imagepull secret in Kubernetes:
    a. `kubectl create secret docker-registry wercker-demo --docker-username=< DOCKERHUB USERNAME > --docker-password="< DOCKERHUB PASSWORD >" --docker-email=< DOCKERHUB EMAIL >`
29. First, start by preparing what we're going to send to Kubernetes, and define how we're going to authenticate by adding the following in wercker.yml

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

30. Next, define the actual Kubectl interaction:

```
    - kubectl:
        name: deploy to kubernetes
        server: $KUBERNETES_MASTER
        certificate-authority: ca.crt
        client-certificate: cert.pem
        client-key: key.pem
        command: apply -f $WERCKER_OUTPUT_DIR/kubernetes/

```

31. Notice that we're referencing more environment variables that will need to be defined:
    a. KUBERNETES_MASTER
    b. KUBERNETES_CA
    c. KUBERNETES_CLIENT_CERT
    d. KUBERNETES_CLIENT_KEY


32. Our Vagrant-launched local Kubernetes cluster is configured with a service account that authenticates via a client key, client certificate, and certificate authority files. Switch to the Kubernetes terminal window and run `kubectl config view`. Under "default-cluster" you should see that these files have been created and written to `artifacts/tls`. We need to pass these certificates to Wercker too, so that it can interact with Kubernetes. Note that Kubernetes has a bunch of different ways to authenticate, we're using the certificate method as it requires no extra setting up/configuration of the cluster, for demo purposes.

33. We need to convert the multi-line certificate files in to a single line string for the environment variables, so run `awk '$1=$1' ORS='\\n' < FILE NAME >` for each.

34. Now it's time to define all of our environment variables. Inside the Wercker HoL VM, create a file called `local.env`, and open it in your text editor. Replace the values as noted:

```
X_IMAGEPULL_SECRET="wercker-demo"
X_DOCKER_REPO="< SAME DOCKER_REPO THAT'S ON WERCKER WEB >"
X_WERCKER_GIT_BRANCH="master"
X_KUBERNETES_MASTER="https://172.17.8.101 < THE KUBERNETES API SHOULD ALWAYS COME UP ON THIS IP USING THIS LOCAL VAGRANT CLUSTER >
X_KUBERNETES_CA=" < SINGLE LINE OUTPUT FROM AWK >"
X_KUBERNETES_CLIENT_CERT="< SINGLE LINE OUTPUT FROM AWK >"
X_KUBERNETES_CLIENT_KEY="< SINGLE LINE OUTPUT FROM AWK >"
```

35. Next, we can try to run our deploy-to-kubernetes pipeline in the Wercker CLI, ensuring we tell it where to find the environment variables it needs: `wercker --environment local.env build --pipeline deploy-to-kubernetes`
36. Jump back in to your terminal window where you launched the Kubernetes VM and run `kubectl get pods` to see if your pods came up successfully.

They should have. In which case, Congrats, you just built your application, created a docker image from the resulting artifact ,and deployed it to a local Kubernetes cluster in an entirely repeatable and independent way using the Wercker CLI.

However, the `kubectl apply` command's success simply shows whether the Kubernetes accepted the instructions we passed to it. It's not telling you whether or not they were successful. This means you had to run `kubectl get pods` outside of your build and deployment flow to find out whether it was successful, which is not ideal since it means we can't fully automate the end-to-end flow.

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

37. Re-run your Kubernetes deployment pipeline in the Wercker CLI with `wercker --environment local.env build --pipeline deploy-to-kubernetes`. The output of the check deployment status step will wait up to 240 seconds for your new Kubernetes pod to become available. If the timeout is hit or the pods fail, the step will fail.

38. For demo purposes, we won't get in to Ingress controllers, instead we rely on Kuberetes NodePorts, which we've defined as `30297`.

39. Lets set up a port forward entry on VirtualBox to forward the Node Port to some local port, which would let you hit the Kubernetes time service locally.
    a. Open the VirtualBox UI.
    b. Select the Master VM
    c. Go to Settings
    d. Network Tab.
    e. Adapter 1
    f. Advanced drop down
    g. Port Forwarding button
    h. Add a new rule with just the host and guest port defined as 30297. Everything else can remain blank (other than Protocol)

40. `curl localhost:30297` from your laptop!

Other than any little hacks for the purpose of running your CI/CD flow and deployment target locally, you have just taken your application from source to deployed on Kubernetes with Wercker!

## Conclusion

Hopefully this has given you an idea of what it takes to take a Java application from source, through tests and on to a Kubernetes cluster with Wercker, along with the option of running the whole process locally.

Comments, suggestions, and improvements are always welcome! :)
