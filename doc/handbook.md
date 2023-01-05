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
- CircleCI uses Helm to upgrade the staging Kubernetes cluster. Kubernetes
  pulls the image from DockerHub.
- The same step if repeated for the prod Kubernetes cluster.

### Project bootstrap

Bootstraping projects require the following permissions:

- DockerHub steps require a DockerHub account which is part of the Jobteaser
  organization. It must be in the `owners` group. There should be at least one
  member of each squad with the required permissions.
- Read Kubernetes secrets require access to the namespace.

A new project must perform the following steps to use the CI/CD pipeline:

- Create a new repository on DockerHub associated with the Github repository
  of the project. Do not initiate a build.
- Give read and write access to the `jobteaser` DockerHub group on the
  DockerHub repository. This will allow CircleCI to push images it builds to
  DockerHub.
- Add the initial CircleCI configuration to the project repository. If using a
  service library such as `go-service` or `rb-service`, the project generator
  already created this configuration. This should be done in a branch.
- Add the repository to CircleCI. Do not initiate a build.
- Add a checkout SSH key in the configuration of the CircleCI project.
- Add the following environment variables to the CircleCI project:
  - `K8S_CA_CERT_STAGING`
  - `K8S_CA_CERT_PROD`
  - `K8S_USER_TOKEN_STAGING`
  - `K8S_USER_TOKEN_PROD`

  CircleCI will need a user token and a CA certificate to connect to staging
  and prod Kubernetes clusters. This shell command helps you configure your
  CircleCI.

  Note: you can use the following commands to extract the token and
  certificate (you'll need
  [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/),
  [kubectx](https://github.com/ahmetb/kubectx) and
  [circleci-env](https://github.com/jobteaser-oss/circleci-env) installed
  first):

  - Create a CircleCI token (https://circleci.com/account/api)
  - Have the
    [operator](https://github.com/jobteaser/service/blob/master/doc/howto.md#how-to-delegate-kubernetes-permissions)
    role on the namespace
  - Run the following command:

        curl -sSL https://raw.githubusercontent.com/jobteaser/circleci/master/utils/configure-project \
            | sh -s -- -t <circleci-token> <service-name>

- If your Kubernetes setup use Vault secrets, make sure these are created
  manually before the first deployment.
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
Dockerfile for the `deploy` [executor](https://github.com/jobteaser/circleci/blob/master/orbs/helm/orb.yml#L7) of the `helm` orb is available at
[_orbs/helm/Dockerfile.deploy_](https://github.com/jobteaser/circleci/blob/master/orbs/helm/Dockerfile.deploy).

### Orbs development

#### Prerequisites

- Clone repo [https://github.com/jobteaser/circleci](https://github.com/jobteaser/circleci)
- Install circleci cli: [https://circleci.com/docs/2.0/local-cli/](https://circleci.com/docs/2.0/local-cli/)
and run `circleci setup` to authenticate yourself via the CLI (you'll need a CircleCI API token)

List of orbs available in JT: [https://circleci.com/developer/orbs?query=jobteaser](https://circleci.com/developer/orbs?query=jobteaser)
File used to illustrate in this doc: [https://github.com/jobteaser/circleci/blob/master/orbs/e2e-web/orb.yml](https://github.com/jobteaser/circleci/blob/master/orbs/e2e-web/orb.yml)

#### Orbs creation guide

If you want to create a new orb:
- On a branch, create a new folder on orbs repository with the name of
your custom orb
- Create a new file _orb.yml_ in your new folder. This file will be your
orb configuration file.

**Example**
To create a new orb `e2e-web`, create a folder called `e2e-web` with an
`orb.yml` file in it:

```
orbs/
├── e2e-web
│   └── orb.yml
...
```

In this `orb.yml` file, define your needed commands, jobs, executor… You can
also use parameters which will be used when you will call your orb.

**Example**

```
jobs:
  execute_e2e_tests:
    parameters:
      tags:
        type: string
        default: "not @flaky and not @wip and not @browserstack and not @prod"
      max_instances:
        type: string
        default: "30"
      command:
        description: "The command to run in e2e_jt_tests repo"
        type: "string"
        default: "npm run chrome"
      bypass_rack_attack:
        description: "The cookie to add to bypass rate limit"
        type: "string"
        default: $BYPASS_RACK_ATTACK
    executor:
      name: "default"
      tags: << parameters.tags >>
      max_instances: << parameters.max_instances >>
    steps:
      - service/configure_ssh
      - clone_test_suite
      - manage_dependencies
      - create_report_folder
      - run_e2e_test:
          command: << parameters.command >>
          bypass_rack_attack: << parameters.bypass_rack_attack >>
      - generate_reporting
      - slack_alerting
```

In the example above, a job called `execute_e2e_tests` is created, with custom
parameters `tags`, `max_instances`, `command` and `bypass_rack_attack`. Note that few
steps used (like `clone_test_suite`, `create_report_folder`, ...) are also custom
commands present in [_orb.yml_](https://github.com/jobteaser/circleci/blob/master/orbs/e2e-web/orb.yml) file.

#### Publishing an orb

When you're done editing your orb, you must publish it on CircleCI to be able
to use it. **This also required to just test your Orb during development.**

If the orb you want to publish is a **new one, you must first create a namespace
for it on CircleCI** (https://circleci.com/docs/2.0/orb-author-intro/#quick-start).

Open your terminal on your circleci repo:
```shell
# Replace `orbName` with the name of your orb (ie your orb folder name).
circleci orb create jobteaser/<orbName>

# Ex: circleci orb create jobteaser/e2e-web
```

**IMPORTANT: you need admin rights on CircleCI to be able to create a new orb
namespace**. If you don't have such rights, ask **DevEx** team to do it for you
(you'll get a `Error: AUTHORIZATION_FAILURE` if you have missing rights).

**To test your orb, you must then publish it** with a custom tag to be used in your
other repo which will use that orb.

```shell
# Replace `orbName` with the name of your orb (ie your orb folder name).
# Replace `any_name_without_slashes` with any name you'd like (just avoid using "/" in the name)
circleci orb publish orbs/<orbName>/orb.yml jobteaser/<orbName>@dev:<any_name_without_slashes>

# Ex: circleci orb publish orbs/e2e-web/orb.yml jobteaser/e2e-web@dev:expose-new-awesome-job
```

Result:

```
Orb jobteaser/circleci@dev:expose-new-awesome-job was published.
Please note that this is an open orb and is world-readable.
Note that your dev label `dev:expose-new-awesome-job` can be overwritten by anyone in your organization.
Your dev orb will expire in 90 days unless a new version is published on the label `dev:expose-new-awesome-job`.

```

It will generate a temporary orb that can be overwritten if you republished, and will be
deleted after 90 days if not.

#### Testing an orb

To test in another repo an orb that you published, declare it in the `.circleci/config.yml`
file of that repo:

```
orbs:
  e2e-web: "jobteaser/e2e-web@dev:expose-new-awesome-job"
```

You can now call your custom commands/jobs:

```
- e2e-web/execute_e2e_tests:
          name: "test_e2e"
          requires: ["deploy_feature_env"]
          filters:
            branches:
              ignore:
                - master
          # Orb custom parameters:
          tags: "not @legacy and not @flaky and not @pending and not @browserstack and not @prod"
          max_instances: "30"
          command: FEATURE_UI=$CIRCLE_SHA1 npm run chrome
          bypass_rack_attack: $BYPASS_RACK_ATTACK
```

**IMPORTANT: Do not forget to prefix your command / job with the name of your orb**
(`e2e-web/execute_e2e_tests` and not just `execute_e2e_tests`).

Once your orb is working as expected, you can **open a PR on `circleci` repo**.
When merged, tag the master with a new version (check current tag and increment):
```
git tag vX.Y.Z
git push origin --tags
```

This will trigger the deployment of a tagged version of your Orb on CircleCI:
https://app.circleci.com/pipelines/github/jobteaser/circleci.

When done, you can **modify your `.circleci/config.yml`** to use the final & versionned orb:
```
orbs:
  e2e-web: "jobteaser/e2e-web@0.11.0"
```

#### Building & using a new docker image in an orb

See https://github.com/jobteaser/circleci/blob/master/doc/handbook.md#executors for
how to name the Dockerfile.

Dockerhub has a feature called autobuild that let's you configure hooks to have dockerhub build docker images when you push to a repository.

If your docker image has no confidential information stored in it, you can define it as being public (like circleci-helm-deploy).

To create a dockerhub repository that will hold the built images, you must:
1. go to: https://hub.docker.com/repository/create to create a "repository"
2. Use those informations:
```
Organization: jobteaser
Repo name: <repo-orbname-executorname>
Visibility: Public (if possible)
Build settings: Connected
Github details: org "jobteaser", repo "circleci", branch "master"
Dockerfile location: /orbs/<orbName>/Dockerfile.<executor_name>
Enable build caching: true
```

**If Dockerfile is not yet present on master, click "Create" and NOT "Create & Build".**

**IMPORTANT: Everytime you make a change to your docker image content, you must build and push the built image to Dockerhub to be able to test it in your orb.**

Use those commands to build, tag & push image to Dockerhub:
```sh
# Build
docker build orbs/<orbName>/ -f orbs/<orbName>/Dockerfile.<executorName> -t <repo-orbname-executorname>:0.0.1

# Tag
# docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
docker tag <repo-orbname-executorname>:0.0.1 jobteaser/<repo-orbname-executorname>:0.0.1

# Push
docker push jobteaser/<repo-orbname-executorname>:0.0.1
```

**Wrapping up, here are the steps to follow when developing an orb with a custom docker image:**

1. Edit your files in `orb/<orbname>/*`
2. If your orb does not exist yet, you must first create it via `circleci orb create jobteaser/<orb-name>`
3. If the update is only on `.yml` file, you can just publish the new orb via:
`circleci orb publish orbs/<orbname>/orb.yml jobteaser/<orbname>@dev:<a_name_like_a_branch_without_slash>`
(see below for more details)
4. If the udpate is on other files like dockerfile or a bash / ruby / whatever script, the update must be bundled in a new docker image and pushed to Dockerhub to be tested
```bash
# Build image locally
# docker build orbs/circleci/ -f <path_to_dockerfile> -t <image_name:image_tag>
docker build orbs/<orbName>/ -f orbs/<orbName>/Dockerfile.<executorName> -t <repo-orbname-executorname>:0.0.1
# docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]
docker tag <repo-orbname-executorname>:0.0.1 jobteaser/<repo-orbname-executorname>:0.0.1
# Push image tagged (using TARGET_IMAGE[:TAG] name)
docker push jobteaser/<repo-orbname-executorname>:0.0.1
```
5. Update `orbs/<orbname>/orb.yml` file to specify new docker image tag
```yaml
executors:
  concurrent:
    docker:
      - image: "jobteaser/<repo-orbname-executorname>:0.0.1"
```
6. Publish the new orb file
`circleci orb publish orbs/<orbname>/orb.yml jobteaser/<orbname>@dev:<a_name_like_a_branch_without_slash>`
7. Go to your repository where you want to use the new orb and specify the new version of the orb
```yaml
orbs:
  # circleci: "jobteaser/circleci@dev:<a_name_like_a_branch_without_slash>"
  circleci: "jobteaser/circleci@dev:feat-extract-circleci-concurrent-job-mgt-11"
```
8. Then push your branch on your repo using the orb and validate on CI that everything works as expected