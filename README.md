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

We need:

* Presentation
* Minikube
    * https://kubernetes.io/docs/tutorials/stateless-application/hello-minikube/

* Wercker CLI
* VM to work from:
    * VMWare
    * Virtualbox

* Cool ass opening slide?

- Presentation
- Sign up for Wercker
- Take an application in Github that's broken
- Fork it
- Explain Wercker yml
- Add a dev pipeline
- Run it in the wercker CLI
- It doesn't work
- Fix it
- Now it works
- Cool, now we want to create a docker image with our shit neatly tied up with a little bow.
- Write a simple docker image step
- Use wercker dev --docker-local and check that the image ends up local, but we know the CI/CD part works
- Push to Quay
- Check out the image on quay.
- Set up minikube (how do we auth?)
- Show the pre-built Service and Deployment configuration
- Deploy it to Minikube
- W00t it works and we got feedback right in Wercker but how do we get to it?
- Install an ingress controller
- Write an ingress rule and re-deploy to Kube
- Get to it via Curl



