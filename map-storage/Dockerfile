# The building of ProtoBuf "messages" must be done out of Docker because grpc-node does not ship with ARM64 binaries.
# See: https://github.com/grpc/grpc-node/issues/1405
# When the issue above is closed, we can move back messages building inside Dockerfile

# protobuf build
FROM --platform=$BUILDPLATFORM node:18.12.1-buster-slim@sha256:62798b0191c85e2fc5f06897e345a6c9e8902ea3beedcb8e07463c665c37526d as proto-builder
WORKDIR /usr/src
COPY messages/package-lock.json messages/package.json ./
RUN npm install
COPY messages .
COPY libs ./../libs
RUN npm run tag-version && npm run ts-proto

# final production image
FROM node:18.12.1-buster-slim@sha256:62798b0191c85e2fc5f06897e345a6c9e8902ea3beedcb8e07463c665c37526d
WORKDIR /usr/src
RUN apt-get update && apt-get install -y git
COPY .yarn/releases .yarn/releases
COPY .yarn/plugins .yarn/plugins
COPY package.json yarn.lock .yarnrc.yml ./
COPY map-storage/package.json map-storage/package.json
COPY libs/messages/package.json libs/messages/package.json
COPY libs/map-editor/package.json libs/map-editor/package.json
COPY libs/math-utils/package.json libs/math-utils/package.json
COPY libs/tailwind/package.json libs/tailwind/package.json
RUN yarn workspaces focus map-storage --production
COPY libs ./libs
COPY --from=proto-builder /usr/libs/messages/src ./libs/messages/src
COPY --chown=node:node map-storage map-storage

WORKDIR /usr/src/map-storage

USER node
CMD ["yarn", "run", "start"]
