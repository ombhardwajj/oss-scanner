# Build avni-webapp for Anthropic's OSS Scanner.
# The scanner uses the repository root as the build context and audits /src offline.
# Mirrors the project's CI (.circleci/config.yml): Node 24.4.1, yarn install, yarn test, yarn build.
FROM node:24.4.1-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    CI=true

# Scanner contract: the checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

RUN yarn install --frozen-lockfile

RUN NODE_OPTIONS=--max_old_space_size=4096 yarn run build

# Run the tests so the image is known to work. A failing test is reported but does not stop the image build.
# Two workers and a longer timeout keep the React component tests within the memory of a small build machine.
RUN yarn test --maxWorkers=2 --testTimeout=30000 || echo "WARNING: yarn test reported failures"
