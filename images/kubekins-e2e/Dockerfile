# Copyright 2016 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# This file creates a build environment for building and running kubernetes
# unit and integration tests

FROM gcr.io/k8s-staging-test-infra/bootstrap:v20250702-52f5173c3a

# hint to kubetest that it is in CI
ENV KUBETEST_IN_DOCKER="true"

# Go standard envs
ENV GOPATH /go
ENV PATH /usr/local/go/bin:$PATH

RUN mkdir -p /go/bin
ENV PATH $GOPATH/bin:$PATH

# setup k8s repo symlink
RUN mkdir -p /go/src/k8s.io/kubernetes \
    && ln -s /go/src/k8s.io/kubernetes /workspace/kubernetes

# preinstall:
# - graphviz package for graphing profiles
# - bc for shell to junit
# - rpm for building RPMs with Bazel
RUN apt-get update && \
    apt-get install -y bc \
    graphviz \
    rpm && \
    rm -rf /var/lib/apt/lists/*
# preinstall this for kops tests xref kubetest prepareAws(...)
# TODO(krzyzacy,justisb,chrislovecnm): remove this
RUN pip install --no-cache-dir --break-system-packages awscli

# install cfssl to prevent https://github.com/kubernetes/kubernetes/issues/55589
# The invocation at the end is to prevent download failures downloads as in the bug.
# TODO(porridge): bump CFSSL_VERSION to one where cfssljson supports the -version flag and test it as well.
ARG CFSSL_VERSION
RUN wget -q -O cfssl "https://pkg.cfssl.org/${CFSSL_VERSION}/cfssl_linux-amd64" && \
    wget -q -O cfssljson "https://pkg.cfssl.org/${CFSSL_VERSION}/cfssljson_linux-amd64" && \
    chmod +x cfssl cfssljson && \
    mv cfssl cfssljson /usr/local/bin && \
    cfssl version

# replace kubectl with one from K8S_RELEASE
ARG K8S_RELEASE=latest
RUN rm -f $(which kubectl) && \
    export KUBECTL_VERSION=$(curl -L https://dl.k8s.io/release/${K8S_RELEASE}.txt) && \
    wget https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl && \
    chmod +x /usr/local/bin/kubectl

# install go
ARG GO_VERSION
ENV GO_TARBALL "go${GO_VERSION}.linux-amd64.tar.gz"
RUN wget -q "https://go.dev/dl/${GO_TARBALL}" && \
    tar xzf "${GO_TARBALL}" -C /usr/local && \
    rm "${GO_TARBALL}"

# install yq
ARG YQ_VERSION
RUN wget -q "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64" \
    -O /usr/local/bin/yq && chmod +x /usr/local/bin/yq

# install bazel
ARG BAZEL_VERSION_ARG
ENV BAZEL_VERSION=${BAZEL_VERSION_ARG}
COPY images/kubekins-e2e/install-bazel.sh /
RUN bash /install-bazel.sh

# install kind if a version is provided
ARG KIND_VERSION
RUN if [ -n "${KIND_VERSION}" ]; then \
    wget -q -O /usr/local/bin/kind https://kind.sigs.k8s.io/dl/v${KIND_VERSION}/kind-linux-amd64 && \
    chmod +x /usr/local/bin/kind; \
    fi

# install kubetest2 binaries if a version is provided
# kubetest2 must be installed from git as the makefile embeds the git short hash and the day it was built.
ARG KUBETEST2_VERSION
RUN if [ -n "${KUBETEST2_VERSION}" ]; then \
    git clone https://github.com/kubernetes-sigs/kubetest2.git /tmp/kubetest2 && \
    cd /tmp/kubetest2 && make install-all && rm -rf /tmp/kubetest2; \
    fi

# configure dockerd to use mirror.gcr.io
# per instructions at https://cloud.google.com/container-registry/docs/pulling-cached-images
ARG DOCKER_REGISTRY_MIRROR_URL=https://mirror.gcr.io
RUN [ -n "${DOCKER_REGISTRY_MIRROR_URL}" ] && \
    echo "DOCKER_OPTS=\"\${DOCKER_OPTS} --registry-mirror=${DOCKER_REGISTRY_MIRROR_URL}\"" | \
    tee --append /etc/default/docker

# add env we can debug with the image name:tag
ARG IMAGE_ARG
ENV IMAGE=${IMAGE_ARG}

# everything below will be triggered on every new image tag ...
ADD ["images/kubekins-e2e/kops-e2e-runner.sh", \
    "images/kubekins-e2e/kubetest", \
    "https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/get-kube.sh", \
    "logexporter/cluster/log-dump.sh", \
    "logexporter/cluster/logexporter-daemonset.yaml", \
    "/workspace/"]
ENV LOG_DUMP_SCRIPT_PATH "/workspace/log-dump.sh"
RUN ["chmod", "+x", "/workspace/get-kube.sh", "/workspace/log-dump.sh"]

