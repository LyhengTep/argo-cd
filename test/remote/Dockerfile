ARG BASE_IMAGE=docker.io/library/ubuntu:24.04@sha256:80dd3c3b9c6cecb9f1667e9290b3bc61b78c2678c02cbdae5f0fea92cc6734ab

FROM docker.io/library/golang:1.24.3@sha256:1bcf8844d2464a6485c87646f9da684610758eb1a2df63c8a6e7ca47c64f8655 AS go

RUN go install github.com/mattn/goreman@latest && \
    go install github.com/kisielk/godepgraph@latest

FROM $BASE_IMAGE

ENV DEBIAN_FRONTEND=noninteractive
RUN  apt-get update && apt-get install --no-install-recommends -y \
    ca-certificates \
    curl \
    openssh-server \
    nginx \
    fcgiwrap \
    git \
    git-lfs \
    gpg \
    make \
    wget \
    gcc \
    sudo \
    zip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# These are required for running end-to-end tests
COPY ./test/fixture/testrepos/id_rsa.pub /root/.ssh/authorized_keys
COPY ./test/fixture/testrepos/nginx.conf /etc/nginx/nginx.conf
COPY ./test/fixture/testrepos/.htpasswd /etc/nginx/.htpasswd
COPY ./test/fixture/testrepos/sudoers.conf /etc/sudoers
COPY ./test/fixture/testrepos/ssh_host_*_key* /etc/ssh/
COPY ./test/fixture/certs/argocd-e2e-server.crt /etc/certs/argocd-test-server.crt
COPY ./test/fixture/certs/argocd-e2e-server.key /etc/certs/argocd-test-server.key
COPY ./test/fixture/certs/argocd-test-ca.crt /etc/certs/argocd-test-ca.crt

# Entrypoint is required for container's user management
COPY ./test/remote/entrypoint.sh /usr/local/bin
COPY ./test/remote/Procfile /Procfile

# We need goreman
COPY --from=go /go/bin/goreman /usr/local/bin/goreman

COPY ./test/e2e/testdata/ /app/config/testdata/

# Prepare user configuration & build environments
RUN useradd -l -u 1000 -d /home/user -s /bin/bash user && \
    echo "user ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/user && \
    mkdir -p /home/user && \
    HOME=/home/user git config --global user.name "ArgoCD Test User" && \
    HOME=/home/user git config --global user.email "noreply@example.com" && \
    mkdir -p /var/run/sshd && \
    mkdir -p /root/.ssh && \
    chown -R user /home/user && \
    chgrp -R user /home/user && \
    chown root /etc/ssh/ssh_host_*_key* && \
    chmod 0600 /etc/ssh/ssh_host_*_key  && \
    mkdir -p /tmp/argo-e2e/testdata.git && \
    cd /tmp/argo-e2e/testdata.git && \
    HOME=/home/user git init --bare && \
    cd /app/config/testdata && \
    HOME=/home/user git init && \
    HOME=/home/user git add . && \
    HOME=/home/user git commit -a -m "Initial commit" && \
    HOME=/home/user git remote add origin file:///tmp/argo-e2e/testdata.git && \
    HOME=/home/user git push origin master && \
    mkdir -p /tmp/argo-e2e/submodule.git && \
    cd /tmp/argo-e2e/submodule.git && \
    HOME=/home/user git init --bare && \
    mkdir -p /tmp/argo-e2e/submoduleParent.git && \
    cd /tmp/argo-e2e/submoduleParent.git && \
    HOME=/home/user git init --bare && \
    chown -R user /tmp/argo-e2e/testdata.git && \
    chown -R user /tmp/argo-e2e/submodule.git && \
    chown -R user /tmp/argo-e2e/submoduleParent.git && \
    ln -s /usr/libexec/git-core /usr/lib/git-core

RUN echo "[http]" >> /tmp/argo-e2e/testdata.git/config && \
    echo "  receivepack = true" >> /tmp/argo-e2e/testdata.git/config 
RUN echo "[http]" >> /tmp/argo-e2e/submodule.git/config && \
    echo "  receivepack = true" >> /tmp/argo-e2e/submodule.git/config 
RUN echo "[http]" >> /tmp/argo-e2e/submoduleParent.git/config && \
    echo "  receivepack = true" >> /tmp/argo-e2e/submoduleParent.git/config 

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
