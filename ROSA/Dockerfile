FROM gcr.io/projectsigstore/cosign:v1.13.0 as cosign-bin

FROM registry.access.redhat.com/ubi8:8.7-1054.1675788412
COPY --from=cosign-bin /ko-app/cosign /usr/local/bin/cosign
ENTRYPOINT [ "cosign" ]
