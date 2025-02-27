# Copyright 2018 Google, Inc. All rights reserved.
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

FROM golang:1.20 AS builder
WORKDIR /src

# This arg is passed by docker buildx & contains the target CPU architecture (e.g., amd64, arm64, etc.)
ARG TARGETARCH
ARG TARGETOS

ENV GOARCH=$TARGETARCH
ENV GOOS=$TARGETOS

ENV CGO_ENABLED=0
ENV GOBIN=/usr/local/bin

# Get GCR credential helper
# Until https://github.com/GoogleCloudPlatform/docker-credential-gcr/issues/128 is addressed, we need to use
# the existing compiled versions distributed in GitHub releases
ARG GCR_HELPER_VERSION="2.1.8"

# Checksums can be obtained from https://github.com/GoogleCloudPlatform/docker-credential-gcr/releases/download/v2.1.8/checksums.txt
ARG GCR_HELPER_SHA_linux_amd64="0da9a0878d34dc8b38ae76519b7137f3179f573cf9c84fec1b81e973b565e2f9"
ARG GCR_HELPER_SHA_linux_386="2421d3242beb239eb640daf9dbfa4582416049a92ed93a207a0a3102fde4ea11"
ARG GCR_HELPER_SHA_linux_s390x="5d18e62c5952d43993b7ee292f87d812ccd161d4e9d932fbc1567eb5d5013632"
ARG GCR_HELPER_SHA_linux_arm64="9d0e67c18fb3e1e048120d06edb9e54e1c11f88139ec77dbc9830e8a8b31745c"

# There is no linux/ppc64le available for docker-credential-gcr, at least let's install it using go install pointing to the tag's commit
RUN if [ "$TARGETARCH" = "ppc64le" ]; then \
      GCR_HELPER_COMMIT=$(git ls-remote -t https://github.com/GoogleCloudPlatform/docker-credential-gcr v$GCR_HELPER_VERSION | head -1 | awk '{print $1}') \
      && go install github.com/GoogleCloudPlatform/docker-credential-gcr@${GCR_HELPER_COMMIT}; \
    else \
      curl -fsSL -o /tmp/gcr-helper.tar.gz https://github.com/GoogleCloudPlatform/docker-credential-gcr/releases/download/v${GCR_HELPER_VERSION}/docker-credential-gcr_${TARGETOS}_${TARGETARCH}-${GCR_HELPER_VERSION}.tar.gz \
      && eval "GCR_HELPER_SHA=\${GCR_HELPER_SHA_${TARGETOS}_${TARGETARCH}}" \
      && echo "${GCR_HELPER_SHA}  /tmp/gcr-helper.tar.gz" | sha256sum -c - \
      && tar -xzf /tmp/gcr-helper.tar.gz --directory /usr/local/bin/; \
    fi

# Get Amazon ECR credential helper
RUN go install github.com/awslabs/amazon-ecr-credential-helper/ecr-login/cli/docker-credential-ecr-login@adf1bafd791ae7d4ff098108b1e91f36a4da5404 # v0.7.1

# Get ACR docker env credential helper
RUN go install github.com/chrismellard/docker-credential-acr-env@82a0ddb2758901b711d9d1614755b77e401598a1

# Add .docker config dir
RUN mkdir -p /kaniko/.docker

COPY . .
RUN \
  --mount=type=cache,target=/root/.cache/go-build \
  --mount=type=cache,target=/go/pkg \
  make out/executor out/warmer

# Generate latest ca-certificates
FROM debian:bullseye-slim AS certs
RUN apt update && apt install -y ca-certificates

# use musl busybox since it's staticly compiled on all platforms
FROM busybox:musl AS busybox

FROM scratch AS kaniko-base-slim

# Create kaniko directory with world write permission to allow non root run
RUN --mount=from=busybox,dst=/usr/ ["busybox", "sh", "-c", "mkdir -p /kaniko && chmod 777 /kaniko"]

COPY --from=certs /etc/ssl/certs/ca-certificates.crt /kaniko/ssl/certs/
COPY files/nsswitch.conf /etc/nsswitch.conf
ENV HOME /root
ENV USER root
ENV PATH /usr/local/bin:/kaniko
ENV SSL_CERT_DIR=/kaniko/ssl/certs

FROM kaniko-base-slim AS kaniko-base

COPY --from=builder --chown=0:0 /usr/local/bin/docker-credential-gcr /kaniko/docker-credential-gcr
COPY --from=builder --chown=0:0 /usr/local/bin/docker-credential-ecr-login /kaniko/docker-credential-ecr-login
COPY --from=builder --chown=0:0 /usr/local/bin/docker-credential-acr-env /kaniko/docker-credential-acr-env

COPY --from=builder /kaniko/.docker /kaniko/.docker

ENV DOCKER_CONFIG /kaniko/.docker/
ENV DOCKER_CREDENTIAL_GCR_CONFIG /kaniko/.config/gcloud/docker_credential_gcr_config.json
WORKDIR /workspace

### FINAL STAGES ###

FROM kaniko-base AS kaniko-warmer

COPY --from=builder /src/out/warmer /kaniko/warmer

ENTRYPOINT ["/kaniko/warmer"]

FROM kaniko-base AS kaniko-executor

COPY --from=builder /src/out/executor /kaniko/executor

ENTRYPOINT ["/kaniko/executor"]

FROM kaniko-executor AS kaniko-debug

ENV PATH /usr/local/bin:/kaniko:/busybox

COPY --from=builder /src/out/warmer /kaniko/warmer

COPY --from=busybox /bin /busybox
# Declare /busybox as a volume to get it automatically in the path to ignore
VOLUME /busybox

RUN ["/busybox/mkdir", "-p", "/bin"]
RUN ["/busybox/ln", "-s", "/busybox/sh", "/bin/sh"]

FROM kaniko-base-slim AS kaniko-slim

COPY --from=builder /src/out/executor /kaniko/executor

ENTRYPOINT ["/kaniko/executor"]
