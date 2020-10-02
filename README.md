# gcsfuse-tcpdump-sidecar
You can use this image as a sidecar in your deployment to store pod tcpdumps to google bucket

Image:
https://hub.docker.com/r/binary000/gcsfuse-tcpdump-sidecar

Requirements:
* Container needs to run as privileged container

required environment variables:
*   **SERVICE_ACCOUNT_KEY** - base64 encoded json key of service account with permissions to write to desired bucket (was tested with storage admin rights for the bucket)
*    **BUCKET_NAME** - name of the bucket where tcpdumps stores files


Optional environment variables:

* **TCPDUMP_OPTIONS** - in case you want to overwrite defaults (-Z root -z gzip -G 120 -i any -vvn)
* **DUMP_FILENAME** - in case you want to overwrite defaults ($HOSTNAME-%d_%m_%Y_%H.%M.%S-tcpdump.pcap)

Example of container configuration

```
name: tcpdump-sidecar
image: binary000/gcsfuse-tcpdump-sidecar
lifecycle:
  preStop:
    exec:
      command:
      - fusermount
      - "-u"
      - "/tmp/tcpdumps"
securityContext:
  privileged: true
  capabilities:
    add:
    - SYS_ADMIN
resources:
  requests:
    cpu: '1'
    memory: 1Gi
```

Dockerfile

```
FROM ubuntu
ENTRYPOINT [ "/bin/bash" ]
RUN apt-get update && \
    apt-get install -y tcpdump \
                       lsb-release \
                       gnupg2 \
                       curl && \
    export GCSFUSE_REPO=gcsfuse-`lsb_release -c -s` && \
    echo deb http://packages.cloud.google.com/apt $GCSFUSE_REPO main | tee /etc/apt/sources.list.d/gcsfuse.list && \
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - && \
    apt-get update && \
    apt-get install -y gcsfuse

RUN mkdir /tmp/tcpdumps

ENV TCPDUMP_OPTIONS="-Z root -z gzip -G 120 -i any -vvn"
ENV DUMP_FILENAME="$HOSTNAME-%d_%m_%Y_%H.%M.%S-tcpdump.pcap"

CMD ["-c", "echo $SERVICE_ACCOUNT_KEY | base64 -d > tcpdump-sidecar-key.json && \
            gcsfuse --key-file=tcpdump-sidecar-key.json -o nonempty $BUCKET_NAME /tmp/tcpdumps && \
            tcpdump $TCPDUMP_OPTIONS -w /tmp/tcpdumps/$DUMP_FILENAME"]
```
