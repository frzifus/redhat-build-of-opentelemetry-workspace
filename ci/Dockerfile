# The Dockerfile's resulting image is purpose-built for running this repo's
# skills (`.claude/skills/<name>`) as agentic jobs within the OpenShift release
# (https://github.com/openshift/release) using Prow CI. It bundles the Claude
# Code CLI together with this repo's own content, so a skill-agent step can
# read a skill's SKILL.md directly from the image instead of fetching it over
# the network at runtime.

FROM docker.io/library/golang:1.27@sha256:f44f6e88636cfb311f9ebace870ded69d943f227bb3cb27d32ffd84ea18c43ea

# Copy the repository files so skills under .claude/skills/ are available
# locally to the agent step.
COPY . /tmp/redhat-build-of-opentelemetry-workspace

WORKDIR /tmp/redhat-build-of-opentelemetry-workspace

# Set the Go path and Go cache environment variables
ENV GOPATH=/tmp/go
ENV GOBIN=/tmp/go/bin
ENV GOCACHE=/tmp/.cache/go-build
ENV PATH=$PATH:$GOBIN

# Create the /tmp/go/bin and build cache directories, and grant read and write permissions to all users
RUN mkdir -p /tmp/go/bin $GOCACHE \
    && chmod -R 777 /tmp/go/bin $GOPATH $GOCACHE

# Install dependencies required by skill execution
RUN apt-get update && apt-get install -y --no-install-recommends \
      curl git jq ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install Claude Code CLI via signed apt repository
RUN install -d -m 0755 /etc/apt/keyrings \
    && curl -fsSL https://downloads.claude.ai/keys/claude-code.asc \
         -o /etc/apt/keyrings/claude-code.asc \
    && echo "deb [signed-by=/etc/apt/keyrings/claude-code.asc] https://downloads.claude.ai/claude-code/apt/stable stable main" \
         > /etc/apt/sources.list.d/claude-code.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends claude-code \
    && rm -rf /var/lib/apt/lists/*
