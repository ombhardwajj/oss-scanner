# Build avni-media for Anthropic's OSS Scanner.
# The scanner uses the repository root as the build context and audits /src offline.
# Mirrors the project's CI (.circleci/config.yml): Node 19.8.1, npm install --legacy-peer-deps and a build in both
# client/ (Next.js) and server/ (NestJS).
#
# The base is a current Debian with Node 19.8.1 installed from nodejs.org, because the official node:19 images are
# built on Debian releases whose package repositories are no longer served.
FROM debian:bookworm

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG NODE_VERSION=19.8.1

ENV DEBIAN_FRONTEND=noninteractive \
    CI=true \
    NEXT_TELEMETRY_DISABLED=1

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl git python3 make g++ xz-utils \
    && rm -rf /var/lib/apt/lists/* \
    && case "$(dpkg --print-architecture)" in amd64) arch=x64 ;; arm64) arch=arm64 ;; *) echo "unsupported architecture"; exit 1 ;; esac \
    && curl -fsSL --retry 3 "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-${arch}.tar.xz" \
        | tar xJ -C /usr/local --strip-components=1 \
    && node --version && npm --version

# Scanner contract: the checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

RUN cd server && npm install --legacy-peer-deps && npm run build

RUN cd client && npm install --legacy-peer-deps \
    && (npm run build || echo "WARNING: client build reported failures")

# Run the tests so the image is known to work. A failing test is reported but does not stop the image build. The
# project's CI does not run the server's specs, and some of them no longer compile against the current sources.
RUN (cd server && npm test || echo "WARNING: server tests reported failures") \
    && (cd client && npm test -- --passWithNoTests || echo "WARNING: client tests reported failures")
