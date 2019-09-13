FROM ubuntu:18.04

LABEL vendor="JobTeaser"
LABEL maintainer="opensource@jobteaser.com"

ENV HELM_VERSION 2.12.3
ENV KUBE_VERSION 1.14.5

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update -y && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
      # base
      wget ca-certificates \
      # checkout
      git openssh-client \
    && \
    rm -rf /var/lib/apt/lists/* && \
    ( \
      cd /tmp && \
      wget -q https://storage.googleapis.com/kubernetes-release/release/v$KUBE_VERSION/bin/linux/amd64/kubectl && \
      install -m755 kubectl /usr/local/bin/kubectl \
    ) && \
    ( \
      cd /tmp && \
      wget -q https://get.helm.sh/helm-v$HELM_VERSION-linux-amd64.tar.gz && \
      tar -xzf helm-v$HELM_VERSION-linux-amd64.tar.gz && \
      cd linux-amd64 && \
      install -m755 helm /usr/local/bin/helm && \
      install -m755 tiller /usr/local/bin/tiller \
    ) && \
    \
    apt-get autoremove -y && \
    apt-get clean -y
