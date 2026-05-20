ARG BUILD_FROM
FROM $BUILD_FROM

# GitHub Actions runner binary version. Pinned (no `:latest`) so a rebuild
# is reproducible. To upgrade: check https://github.com/actions/runner/releases
# and bump this ARG. Don't forget to bump config.yaml `version:` too — HA
# only re-pulls when the add-on version changes.
ARG GITHUB_RUNNER_VERSION="2.334.0"

# Why Debian (not Alpine):
# The actions/runner binary embeds a .NET 6 runtime built against glibc.
# Alpine's musl libc is not ABI-compatible — even with gcompat/libc6-compat
# shims the .NET runtime's ICU calls hit ABI mismatches at startup with
# "Error loading shared library ld-linux-x86-64.so.2". GitHub has declined
# to support Alpine (actions/runner#801). Switching to Debian Bookworm
# removes the whole class of breakage at the cost of a larger base image
# (~200MB vs Alpine ~50MB). The runner itself is ~600MB extracted, so the
# base-size delta is noise relative to the install footprint.
#
# Why NodeSource for Node:
# Debian Bookworm's default is Node 18, which doesn't satisfy modern
# package.json engines. Node 20 is current LTS through 2026; NodeSource's
# apt repo provides clean upgrades without compiling from source.
#
# Why wrangler@4 baked into the image:
# Workflows that deploy to Cloudflare (`d1 migrations apply`, `opennextjs-
# cloudflare deploy`) call wrangler on every run. Installing it at image
# build time avoids a ~10s `npm install -g` cold-start per workflow.
#
# Why libsqlite3-0 baked into the image:
# Flutter projects using `drift` (a Dart SQLite ORM) need a host libsqlite3
# at test time — drift's native binding dlopens `libsqlite3.so.0`. Bookworm
# minimal doesn't ship it. Q-Call's `flutter test` was hard-failing on this
# before v3.0.3.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
         ca-certificates curl gnupg \
    && mkdir -p /etc/apt/keyrings \
    && curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key \
         | gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg \
    && echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_20.x nodistro main" \
         > /etc/apt/sources.list.d/nodesource.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
         bash git jq \
         libicu72 libssl3 \
         libsqlite3-0 \
         nodejs \
    && npm install -g wrangler@4 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /var/cache/apt/*

RUN mkdir -p /opt/runner \
    && cd /opt/runner \
    && curl -L -o runner.tar.gz \
         "https://github.com/actions/runner/releases/download/v${GITHUB_RUNNER_VERSION}/actions-runner-linux-x64-${GITHUB_RUNNER_VERSION}.tar.gz" \
    && tar xzf runner.tar.gz \
    && rm runner.tar.gz

COPY rootfs /

RUN chmod -R 0755 /opt/runner \
    && chmod 0755 /etc/services.d/runner/run

# /opt/runner/ is the template installation (read-only at runtime). The
# launch script copies it to /opt/runners/<name>/ per configured runner on
# first start. Don't set WORKDIR — the script does its own per-runner cd.
