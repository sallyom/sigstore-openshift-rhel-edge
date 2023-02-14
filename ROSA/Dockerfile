FROM gcr.io/projectsigstore/cosign:v2.0.0-rc.2 as cosign-bin

FROM registry.access.redhat.com/ubi8:8.7-1054.1675788412
COPY --from=cosign-bin /ko-app/cosign /usr/local/bin/cosign
ENTRYPOINT [ "/bin/sh", "-c" , "cosign initialize --mirror=$TUF_URL --root=$TUF_URL/root.json && sleep 86400" ]
