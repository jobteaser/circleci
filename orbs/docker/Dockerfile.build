FROM ubuntu:18.04

LABEL vendor="JobTeaser"
LABEL maintainer="opensource@jobteaser.com"

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update -y && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
      # base
      wget ca-certificates \
      # checkout
      git openssh-client \
      # build
      docker.io \
      # trivy (note that it also uses git at runtime)
      apt-transport-https gnupg \
    && \
    rm -rf /var/lib/apt/lists/* && \
    ( \
      wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | apt-key add - && \
      echo "deb https://aquasecurity.github.io/trivy-repo/deb bionic main" >>/etc/apt/sources.list.d/trivy.list && \
      apt-get update -y && \
      apt-get install -y trivy && \
      rm -rf /var/lib/apt/lists/* \
    ) && \
    apt-get remove --purge -y \
      wget apt-transport-https gnupg && \
    apt-get autoremove -y && \
    apt-get clean -y
