# Get pre build go lang image from dockerhub.
FROM golang:1.19-alpine3.17 AS build-env

ARG GOPROXY
ENV GOPROXY ${GOPROXY:-direct}

ARG GITEA_VERSION
ARG TAGS="sqlite sqlite_unlock_notify"
ENV TAGS "bindata timetzdata $TAGS"
ARG CGO_EXTRA_CFLAGS

# Get Build deps for compiling.
RUN apk --no-cache add build-base git nodejs npm upx

# Get the source code from github.
RUN git clone https://github.com/go-gitea/gitea.git ${GOPATH}/src/code.gitea.io/gitea
WORKDIR ${GOPATH}/src/code.gitea.io/gitea

# Compile the latest the greatest.
RUN make clean-all build

# Compile the program that set the environment to app.ini file.
RUN go build contrib/environment-to-ini/environment-to-ini.go

# The go programs grows so fast, lets pack the elf file, what i have seen thats a lot, from 100M to 40M.
# And unpack the file into memmory when we run the program.
RUN upx -9 -o /go/src/code.gitea.io/gitea/gitea.upx /go/src/code.gitea.io/gitea/gitea
RUN upx -9 -o /go/src/code.gitea.io/gitea/environment-to-ini.upx /go/src/code.gitea.io/gitea/environment-to-ini

# Lets go and fetch a very small container and put the go compiled files to binary into that
FROM alpine:3.17
LABEL maintainer="maintainers@gitea.io"

EXPOSE 22 3000

# We need some more programs in the container. Like sqlite and git for gitea to work properly.
RUN apk --no-cache add \
    bash \
    ca-certificates \
    curl \
    gettext \
    git \
    linux-pam \
    openssh \
    s6 \
    sqlite \
    su-exec \
    gnupg

# Lets add the group and user git, that runs the binary.
RUN addgroup \
    -S -g 1000 \
    git && \
  adduser \
    -S -H -D \
    -h /data/git \
    -s /bin/bash \
    -u 1000 \
    -G git \
    git && \
  echo "git:*" | chpasswd -e

ENV USER git
ENV GITEA_CUSTOM /data/gitea

VOLUME ["/data"]

ENTRYPOINT ["/usr/bin/entrypoint"]
CMD ["/bin/s6-svscan", "/etc/s6"]

COPY --from=build-env /go/src/code.gitea.io/gitea/docker/root /
COPY --from=build-env /go/src/code.gitea.io/gitea/gitea.upx /app/gitea/gitea
COPY --from=build-env /go/src/code.gitea.io/gitea/environment-to-ini.upx /usr/local/bin/environment-to-ini
RUN chmod 755 /usr/bin/entrypoint /app/gitea/gitea /usr/local/bin/gitea /usr/local/bin/environment-to-ini
RUN chmod 755 /etc/s6/gitea/* /etc/s6/openssh/* /etc/s6/.s6-svscan/*
