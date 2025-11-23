FROM alpine:3.20

# create non-root
RUN addgroup -S nettool && adduser -S -G nettool nettool

WORKDIR /home/nettool

RUN apk add --no-cache \
    bind-tools \
  && rm -rf /var/cache/apk/*

# remove the apk binary to discourage later in-container installs
RUN rm -f /sbin/apk || true

# Reduce SUID/SGID
RUN find / -type f -perm /6000 -exec chmod a-s {} + || true

ENV PATH=/usr/bin:/bin:/usr/sbin:/sbin:/home/nettool:$PATH

USER nettool

ENTRYPOINT ["/bin/sh"]