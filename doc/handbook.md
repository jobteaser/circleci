
# Introduction
This document describes the CI and CD pipelines for JobTeaser services, and
explains how to use and maintain CircleCI orbs.

## CI/CD pipeline

The CI/CD pipeline is made of the following steps:

- Developers push patches to Github.
- Github notifies CircleCI.
- CircleCI run tests and build the project.
- CircleCI creates a Docker image containing the output of the build and push
  it to DockerHub.
- CircleCI uses Helm to upgrade the preprod Kubernetes cluster. Kubernetes
  pulls the image from DockerHub.
- The same step if repeated for the prod Kubernetes cluster.

### Project bootstrap

A new project must perform the following steps to use the CI/CD pipeline:

- Create a new repository on DockerHub associated with the Github repository
  of the project. Do not initiate a build.
- Give read and write access to the `jobteaser` DockerHub group on the
  DockerHub repository. This will allow CircleCI to push images it builds to
  DockerHub.
- Add the initial CircleCI configuration to the project repository. If using a
  service library such as `go-service`, the project generator already created
  this configuration. This should be done in a branch.
- Add the repository to CircleCI. Do not initiate a build.
- Add a checkout SSH key in the configuration of the CircleCI project. This is
  only necessary if the project must access other GitHub repositories, for
  example if it uses private repositories as dependencies.
- Add the following environment variables to the CircleCI project:
  - `K8S_CA_CERT_PREPROD`
  - `K8S_CA_CERT_PROD`
  - `K8S_USER_TOKEN_PREPROD`
  - `K8S_USER_TOKEN_PROD`

  CircleCI will need a user token and a CA certificate to connect to preprod
  and prod Kubernetes clusters. These data are stored in a Kubernetes secret
  of each namespace named `<namespace>-ci-token-token-<id>`. You can use
  `kubectl get secret <name> -o yaml` to obtain the certificate and the token.
  User token environment variables use decoded forms (i.e. use `base64 -d` on
  the content of the secret entry). CA certificates environment variables must
  be base64-encoded.

  Note: you can use the following commands to extract the token and certificate
  (you'll need jq installed first).
  ```sh
  kubectl get secret <secret-name> -o json | jq -r '.data.token' | base64 --decode
  kubectl get secret <secret-name> -o json | jq -r '.data."ca.crt"'
  ```
- Create in both preprod and prod a kubernetes secret for DockerHub
  credentials:
  ```sh
  kubectl create secret docker-registry regcred           \
        --docker-server=https://index.docker.io/v1/       \
        --docker-username=jobteaserfoundation             \
        --docker-password=$DOCKER_PASSWORD                \
        --docker-email=foundation+dockerhub@jobteaser.com
  ```
  The password of the `jobteaserfoundation` DockerHub account is available in
  the chapter backend group of Bitwarden.
- Edit the default service account (`kubectl edit serviceaccount default`) and
  add:
  ```yaml
  imagePullSecrets:
    - name: regcred
  ```
  This will allow Kubernetes to pull images from DockerHub.
- Commit and push your branch. This will trigger the first build.
- Merge the branch in `master`. This will trigger a build and the first
  deployment.

## Orbs
CircleCI offers the possibility to write custom executors, commands and jobs
and make them available in reusable packages called orbs. This mechanism
allows us to define standard JobTeaser procedures, and make them available to
all projects.

Each orb is stored in a subdirectory of the `orbs` top-level directory.

**Note that all orbs are public. They must not include any confidential
information.**

### Versioning
The CircleCI setup for this repository will validate all orbs and publish them
using a development tag based on the git branch, making it easy to test
orbs. The `dev:master` development version can be selected to use the very
last committed version of an orb.

Stable orbs are automatically published for tagged git commits. Tags must
follow the usual `v<major>.<minor>.<patch>` convention. Orbs follow semantic
versioning conventions. Note that for the sake of simplicity, we should
avoid sub-versions (alphas, betas, release candidates, etc.).

CircleCI configuration files can follow major, minor and patch versions;
please be careful not to introduce backward incompatible changes between minor
or patch versions. As a rule of thumb:

- bump the patch version if you fix an issue without changing the interface;
- bump the minor version if you are adding backward compatible features;
- bump the major version if you are making backward incompatible changes.

### Tests
If you need to experiment with orbs, just create a branch and publish
development versions based on this branch. In your project, you can then
depend on `dev:<branch>`.

### Executors
Executors defined in orbs are based on Dockerfiles stored in the orb
directory, using the name of the executor as file extension. For example, the
Dockerfile for the `deploy` executor of the `helm` orb is available at
`orbs/helm/Dockerfile.deploy`.
