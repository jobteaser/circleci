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

- If your Kubernetes setup mounts any secret, make sure these are created
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
Dockerfile for the `deploy` executor of the `helm` orb is available at
`orbs/helm/Dockerfile.deploy`.

### Orbs creation

#### **Prerequisites:**

- Clone repo [https://github.com/jobteaser/circleci](https://github.com/jobteaser/circleci)
- Install circleci cli: [https://circleci.com/docs/2.0/local-cli/](https://circleci.com/docs/2.0/local-cli/)

List of orbs available in JT: [https://circleci.com/developer/orbs?query=jobteaser](https://circleci.com/developer/orbs?query=jobteaser)

File used to illustrate in this doc: [https://github.com/jobteaser/circleci/blob/master/orbs/e2e-web/orb.yml](https://github.com/jobteaser/circleci/blob/master/orbs/e2e-web/orb.yml)

#### **Guide**

- On a branch, create a new folder on orbs repository with the name of your custom orb

- Create a new file _orb.yml_ in your folder. This file will be your orb configuration file.

**Example**: to create a new orb “e2e-web”, create folder “e2e-web” with your orb.yml file

- In this _orb.yml_ file, define your needed commands, jobs, executor… You can also use parameters which will be used when you will call your orb.

**Example:**

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
      e2e_cookie:
        description: "The cookie to add to bypass rate limit"
        type: "string"
        default: $COOKIE_E2E
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
          e2e_cookie: << parameters.e2e_cookie >>
      - generate_reporting
      - slack_alerting
```

In this case, I have created a job, with parameters. Steps I used are also custom commands I have in my [_orb.yml_](https://github.com/jobteaser/circleci/blob/master/orbs/e2e-web/orb.yml) file

When your file is configured, you have to create your orb in the repo. Open your terminal on your circleci repo and type:

```
circleci orb create jobteaser/orbName
```

_orbName_ has to be replaced with your orb name (ie your folder name)

**Example:** circleci orb create jobteaser/e2e-web

If you have: Error: AUTHORIZATION_FAILURE ask **devex** team to do it for you

- To test your orb, you can publish it with a custom name to be used in your other repo which will call the orb.

**Example:**

```
circleci orb publish orbs/<<orbName>>/orb.yml jobteaser/<<orbName>>@dev:orb_e2e_tests
```

Result:

```
Orb jobteaser/circleci@dev:orb_e2e_tests was published.
Please note that this is an open orb and is world-readable.
Note that your dev label `dev:orb_e2e_tests` can be overwritten by anyone in your organization.
Your dev orb will expire in 90 days unless a new version is published on the label `dev:orb_e2e_tests`.

```

It will generate a temporary orb that can be overwritten if you republished, and will be deleted after 90 days if not published

- Now your orb can be used in your other repo. On _conf.yml_ file of your other repo, import your orbs you just published:

```
orbs:
  e2e-web: "jobteaser/e2e-web@dev:orb_e2e_tests"
```

You can now call your custom commands/jobs

**Example:**

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
          e2e_cookie: $COOKIE_E2E
```

Do not forget to prefix your command with the name of your orb

- Once you know your orb is correctly working, push your orb to master and tag master with a new version (check current tag and increment)

- Modify on your _conf.yml_ file your orb version with the real one pushed in master and published

```
orbs:
  e2e-web: "jobteaser/e2e-web@0.11.0"
```
