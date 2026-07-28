# Dockerfile.lumnus — Lumnus in-house self-build of Kiwi TCMS Enterprise (AGPL-3.0+).
#
# Upstream `Dockerfile` requires a PKG_TOKEN for the private index (pkg.kiwitcms.eu) and builds
# FROM the subscriber-only base `hub.kiwitcms.eu/kiwitcms/version`. This variant bypasses BOTH,
# using only public sources — verified 2026-07-28:
#   - the `kiwitcms_enterprise` wheel is built here from THIS AGPL source (stage 1), not pulled;
#   - the base is the PUBLIC CE image `pub.kiwitcms.eu/kiwitcms/kiwi` (16.2, which we already run);
#   - every requirements.txt dep is on public PyPI, so --index-url is pypi.org (no PKG_TOKEN).
# Single `docker build` → fits ci-build (the two-phase-build-gap cure, fork-build-publish.md).
# Enterprise version == 16.2 == the base CE version (checked: `setup.py --version`).

# Global ARG — must precede ALL FROMs to be usable in a stage's FROM line (buildkit ARG-in-FROM rule).
ARG KIWI_VERSION=latest

# ---- stage 1: build the enterprise wheel + the 4 moat deps from PUBLIC GitHub source tags ----
# The moat is ONLY the wheel-publishing lag on public PyPI — the source tags ARE public (AGPL).
# We build them ourselves for self-use (no redistribution of a custom EE build).
FROM python:3.12-slim AS wheelbuild
RUN apt-get update && apt-get install -y --no-install-recommends git && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir build
WORKDIR /src
COPY . /src
RUN python -m build --wheel --outdir /wheels          # → /wheels/kiwitcms_enterprise-16.2-*.whl
# the 4 private-index-only pins, built from their public AGPL source tags
RUN pip wheel --no-deps --no-cache-dir --wheel-dir /wheels \
      git+https://github.com/kiwitcms/github-app@v2.3.0 \
      git+https://github.com/kiwitcms/tenants@v4.7.1 \
      git+https://github.com/kiwitcms/trackers-integration@v1.4.0 \
      git+https://github.com/kiwitcms/python-social-auth-kerberos@v0.3.2

# ---- stage 2: layer enterprise onto the PUBLIC CE image (mirrors upstream Dockerfile, token-free) ----
# Base pulled via docker.io (routes through our Zot mirror — no rate-limit). `pub.kiwitcms.eu` is a
# Docker Hub proxy hit WAN-direct by buildkit → 429; docker.io/kiwitcms/kiwi is the same image, mirrored.
FROM docker.io/kiwitcms/kiwi:${KIWI_VERSION}

USER 0
# public kiwitcms/kiwi base is CentOS Stream 10 with dnf (the subscriber `version` base had microdnf).
RUN dnf -y install augeas-libs krb5-libs psmisc xmlsec1 xmlsec1-openssl && dnf clean all

HEALTHCHECK CMD [ -d /proc/$(cat /tmp/nginx.pid) ] && [ -d /proc/$(cat /tmp/kiwitcms.pid) ]

# override NGINX configuration (upstream shape)
CMD /Kiwi/bin/docker-entrypoint
COPY ./etc/nginx.override /Kiwi/etc/nginx.override
RUN ln -sf /tmp/actual.conf /Kiwi/etc/nginx.conf
ENV NGX_AUTHENTICATED_RATE=300  \
    NGX_AUTHENTICATED_BURST=100 \
    NGX_STATIC_RATE=300         \
    NGX_STATIC_BURST=100        \
    NGX_UPLOADS_RATE=10         \
    NGX_UPLOADS_BURST=10        \
    NGX_ERRORS_RATE=1           \
    NGX_ERRORS_BURST=1          \
    NGX_CSP_SCRIPT_SRC=""       \
    NGX_DENY_INCLUDE="/dev/null"

COPY ./bin/* /Kiwi/bin/
COPY --from=wheelbuild /wheels/ /Kiwi/dist/

# Install the enterprise wheel WE built + resolve its deps from PUBLIC PyPI (no PKG_TOKEN).
RUN pip install --no-cache-dir --only-binary :all: decorator lxml && \
    pip install --no-cache-dir --find-links /Kiwi/dist/ \
        --index-url https://pypi.org/simple/ \
        /Kiwi/dist/kiwitcms_enterprise*.whl

# cosmetic cleanup from upstream (broken CSS / missing source-maps) — tolerant of python-minor drift
RUN rm -rf /venv/lib64/python3.12/site-packages/tcms/node_modules/c3/htdocs/ \
           /venv/lib64/python3.12/site-packages/tcms/node_modules/eonasdan-bootstrap-datetimepicker/build/css/bootstrap-datetimepicker-standalone.css \
           /venv/lib64/python3.12/site-packages/tcms/node_modules/bootstrap-touchspin/demo/ \
           /venv/lib64/python3.12/site-packages/django_tenants/templates/admin/index.html || true
RUN touch /venv/lib64/python3.12/site-packages/tcms/node_modules/bootstrap-slider/dependencies/js/jquery.min.map \
          /venv/lib64/python3.12/site-packages/tcms/node_modules/pdfmake/build/module.mjs.map || true

# collect static files (enterprise apps add static; this is load-bearing, keep strict)
RUN cp /Kiwi/static/ca.crt /Kiwi/ssl/ && \
    /Kiwi/manage.py collectstatic --clear --link --noinput && \
    mv /Kiwi/ssl/ca.crt /Kiwi/static/ && \
    mkdir -p /Kiwi/static/.well-known/acme-challenge/

USER 1001
