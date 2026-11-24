# syntax=docker/dockerfile:1.29.0@sha256:572cebbc2b15e6f84c077742952674889cb63cd5e9b80abdffcbe587b6f19183
FROM postgres:18.7-trixie@sha256:9f4fd4aa918f52ae3d6f3b687f1de90d8f867cd4d520021545a9a50c120eae3b

# Patroni lives in its own venv, installed strictly from the hashed lock. Wheels are preferred; a few
# pure-Python deps only ship source packages, and with no compiler in the image any source package
# needing one fails the build instead of slipping in.
RUN --mount=type=bind,source=requirements.lock,target=/tmp/requirements.lock \
    apt-get update \
 && apt-get install -y --no-install-recommends python3 python3-venv \
 && rm -rf /var/lib/apt/lists/* \
 && python3 -m venv /opt/patroni \
 && /opt/patroni/bin/pip install --no-cache-dir --prefer-binary --require-hashes -r /tmp/requirements.lock \
 && install -d -o postgres -g postgres -m 0700 /var/lib/postgresql/data

# Image tooling, and the declarations of everything no package manager owns: they become an SBOM fragment
# that the image's SBOM scanner merges (hippo/components.yaml).
COPY --chmod=0755 hippo/hippo-manifest /usr/local/bin/hippo-manifest
COPY hippo/sbom-fragment.py /usr/local/lib/hippo/sbom-fragment.py
COPY hippo/components.yaml /usr/local/lib/hippo/components/base.yaml
RUN mkdir -p /usr/share/hippo/sbom \
 && /opt/patroni/bin/python /usr/local/lib/hippo/sbom-fragment.py /usr/local/lib/hippo/components/base.yaml base > /usr/share/hippo/sbom/base.spdx.json \
 && rm -rf /root/.cache /root/.wget-hsts

ENV PATH=/opt/patroni/bin:$PATH
USER postgres
# Patroni owns the Postgres lifecycle; SIGTERM makes it shut Postgres down and release the leader key.
STOPSIGNAL SIGTERM
ENTRYPOINT []
CMD ["patroni", "/etc/patroni.yml"]
