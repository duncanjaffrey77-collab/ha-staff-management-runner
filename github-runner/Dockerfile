ARG BUILD_FROM
FROM $BUILD_FROM

# GitHub Actions runner binary version. Pinned (no `:latest`) so a rebuild
# is reproducible. To upgrade: check https://github.com/actions/runner/releases
# and bump this ARG. Don't forget to bump config.yaml `version:` too — HA
# only re-pulls when the add-on version changes.
ARG GITHUB_RUNNER_VERSION="2.319.1"

# Base packages:
#   - bash + curl + git: required by the runner self-installer + workflows
#   - icu-libs + libstdc++: required by the .NET-based runner host binary
#   - nodejs / npm: Node 22 satisfies package.json engines for staff-management
#                   (actions/setup-node@v4 also works at runtime; pre-installing
#                   keeps cold-start fast)
#   - wrangler@4: Cloudflare CLI for `d1 migrations apply` + `opennextjs-cloudflare
#                 deploy` on master pushes. Master-only paths reach this; the
#                 token comes from repo secrets at workflow time.
#
# Alpine `nodejs` in the main repo tracks the LTS series; on HA base images
# (Alpine 3.19+) that's Node 20. If Node 22 is required and the main repo
# doesn't have it, swap to the `nodejs-current` package or add the edge/main
# repo. The runner's own bootstrap (actions/setup-node@v4) downloads whatever
# Node version the workflow asks for, so the system Node here is a convenience,
# not a hard requirement.
RUN apk add --no-cache \
      bash curl git icu-libs libstdc++ \
      nodejs npm \
    && npm install -g wrangler@4 \
    && rm -rf /var/cache/apk/*

RUN mkdir -p /opt/runner \
    && cd /opt/runner \
    && curl -L -o runner.tar.gz \
         "https://github.com/actions/runner/releases/download/v${GITHUB_RUNNER_VERSION}/actions-runner-linux-x64-${GITHUB_RUNNER_VERSION}.tar.gz" \
    && tar xzf runner.tar.gz \
    && rm runner.tar.gz

COPY rootfs /

RUN chmod -R 0755 /opt/runner \
    && chmod 0755 /etc/services.d/runner/run

WORKDIR /opt/runner
