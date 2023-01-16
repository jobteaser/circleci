# Table of contents
1. [Docker registry](#docker-registry)
2. [Kubernetes configuration](#kubernetes-configuration)
3. [Checkout key](#checkout-key)
4. [Enable project](#enable-project)

# Introduction
This document explains how to bootstrap a continuous integration/deployment
from scratch.

# Docker registry
**This step requires a [Dockerhub](https://hub.docker.com) account which is
part of the Jobteaser organization. It must be a `owner` of the
organization. There should be at least one member of each squad with the
required permissions.**

Create a new private Dockerhub repository using the name of the Github
repository. Grant `Read & Write` permission to the `jobteaser` team. This
will allow CircleCI to push the images it builds to Dockerhub.

# Kubernetes configuration
This step requires:
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl) installed
  and
  [configured](https://github.com/jobteaser/service/blob/master/doc/howto.md#how-to-login-on-the-kubernetes-cluster)
  for staging and production Kubernetes cluster.
- [kubectx](https://github.com/ahmetb/kubectx) installed.
- [circleci](https://circleci.com/docs/2.0/local-cli/#installation) installed
  and [configured](https://circleci.com/account/api).
- [circleci-env](https://github.com/jobteaser-oss/circleci-env) installed.
- [operator
  role](https://github.com/jobteaser/kubernetes-namespaces/blob/master/doc/handbook.md#access-management)
  for staging and production.

You have to execute this script:

```sh
curl -sSL https://raw.githubusercontent.com/jobteaser/circleci/master/utils/configure-project \
    | sh -s -- <service-name>
```

The `configure-project` script populates the CircleCI project with
`K8S_USER_TOKEN_STAGING` and `K8S_USER_TOKEN_PROD` environment variable.
There variables are used to etablish an authenticated connection to the
Kubernetes cluster. Don't forget to use the right context `deploy_staging` / 
`deploy_prod` in your CircleCI deployment job.

# Checkout key
By default CircleCI adds a default checkout key with read only permission. The
generated pipeline uses the [Helm
orb](https://github.com/jobteaser/circleci/blob/master/orbs/helm/orb.yml)
which requires write permission to create a Git tag.

You can configure a SSH key with write permission as follows:
1. Connect to GitHub with circleci-jt user [(You should have given push access on your repo to 
   circleci-jt)](https://github.com/jobteaser/service/blob/master/doc/bootstrap.md#repository).
   Credentials are in BitWarden with the name 'GitHub user key for CircleCI'.
2. Then, use this GitHub session to connect to CircleCI
3. Go to your project. Then, on the "project settings > Checkout SSH keys" page, click the
   "Authorize with Github" button. This gives CircleCI permission to
   create and upload SSH keys to Github. If a user key associated with a person profile is already present, replace it with the machine user key. 
4. Click the "Create and add XXX user key" button.

Now CircleCI will use the SSH key with read and write permissions.

# Activate the project
**You must have finished all previous steps before activating your CircleCI
project.**

You can activate your CircleCI project as follows:
1. On the "Add project" page, search for your project.
2. Click the "Set up project" button.

After this part, you should be able to start developing your project.
