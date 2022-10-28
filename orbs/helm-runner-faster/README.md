# Helm-runner

## Circleci runner
The circleci custom runner image is located in a private repo.

## Orb use
In order to use this orb in a job, make sure **not having** any `docker` or
`resource-class` entries, as it will override this orb behavior and the
deployment will fail.
