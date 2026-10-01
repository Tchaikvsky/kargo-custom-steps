#Grype version example. Check for latest version in https://github.com/anchore/grype/release
ARG GRYPE_VERSION=0.110.0

# Extract the grype binary from the distroless official image
FROM anchore/grype:v${GRYPE_VERSION} AS grype

# Use alpine as the base since the official grype image is distroless (no shell)
FROM alpine:3.23.3

RUN adduser -D -u 65532 nonroot && \
    mkdir -p /home/nonroot/

COPY --from=grype /grype /usr/local/bin/grype
COPY grype.tmpl /etc/grype/grype.tmpl
COPY grype.yaml /home/nonroot/.grype.yaml

USER 65532

# Pre-download the vulnerability database
RUN grype db update
