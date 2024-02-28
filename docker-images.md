# Images

## How to pull images from docker hub

First, go to hub.docker.com and you can find images you'd like to pull.

To see images you have installed locally, use `docker image ls`

```
$ docker image ls

REPOSITORY    TAG          IMAGE ID       CREATED         SIZE
sample        verseion-2   f192e6b444a1   7 minutes ago   187MB
nginx         latest       eb4a57159180   12 days ago     187MB
hello-world   latest       9c7a54a9a43c   7 weeks ago     13.3kB
```

This will list all the images along with their sizes. If the images have the same image ID (SHA256), it will just add a new item but won't install again in your local machine.

```
$ docker image pull nginx:1.25.1

1.25.1: Pulling from library/nginx
Digest: sha256:593dac25b7733ffb7afe1a72649a43e574778bf025ad60514ef40f6b5d606247
Status: Downloaded newer image for nginx:1.25.1
docker.io/library/nginx:1.25.1
```

```
$ docker image ls

REPOSITORY    TAG          IMAGE ID       CREATED         SIZE
sample        verseion-2   f192e6b444a1   9 minutes ago   187MB
nginx         1.25.1       eb4a57159180   12 days ago     187MB
nginx         latest       eb4a57159180   12 days ago     187MB
hello-world   latest       9c7a54a9a43c   7 weeks ago     13.3kB
```

You see `nginx:latest` and `nginx:1.25.1` have the same image id and avoided downloading it again on the pull.

## Change the image tag

To create a copy of an image and name it to how you want, you can use `docker image tag`

```
$ docker image tag nginx:latest daniels-random/image-tag:hahaha
```

```
$ docker image ls

REPOSITORY                 TAG          IMAGE ID       CREATED          SIZE
sample                     verseion-2   f192e6b444a1   11 minutes ago   187MB
nginx                      1.25.1       eb4a57159180   12 days ago      187MB
nginx                      latest       eb4a57159180   12 days ago      187MB
daniels-random/image-tag   hahaha       eb4a57159180   12 days ago      187MB
hello-world                latest       9c7a54a9a43c   7 weeks ago      13.3kB
```

This is especially helpful if you are pushing the image to your repository. Create your custom image and use

```
$ docker image tag created-image:tag <user-name>/<repository-name>:<tag>
```

Now you can push it using `docker image push`

```
$ docker image push <image>:<tag>
```

But ensure that you are signed into the docker hub. Use `docker login` to log into the hub.

```
$ docker login
Authenticating with existing credentials...
Login Succeeded

Logging in with your password grants your terminal complete access to your account.
For better security, log in with a limited-privilege personal access token. Learn more at https://docs.docker.com/go/access-tokens/
```

This information gets stored under `.docker/config.json`.

## Building images

We will be building an image. You always need to have a build file, by default uses `Dockerfile` (but you can declare the build file to build using -f).

```Dockerfile
# NOTE: this example is taken from the default Dockerfile for the official nginx Docker Hub Repo
# https://hub.docker.com/_/nginx/
# NOTE: This file is slightly different than the video, because nginx versions have been updated
#       to match the latest standards from docker hub... but it's doing the same thing as the video
#       describes

FROM debian:bullseye-slim
# all images must have a FROM
# usually from a minimal Linux distribution like debian or (even better) alpine
# if you truly want to start with an empty container, use FROM scratch

LABEL maintainer="NGINX Docker Maintainers <docker-maint@nginx.com>"

# optional environment variable that's used in later lines and set as envvar when container is running
ENV NGINX_VERSION   1.23.4
ENV NJS_VERSION     0.7.11
ENV PKG_RELEASE     1~bullseye


RUN set -x \
# create nginx user/group first, to be consistent throughout docker variants
    && addgroup --system --gid 101 nginx \
    && adduser --system --disabled-login --ingroup nginx --no-create-home --home /nonexistent --gecos "nginx user" --shell /bin/false --uid 101 nginx \
    && apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y gnupg1 ca-certificates \
    && \
    NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \
    NGINX_GPGKEY_PATH=/usr/share/keyrings/nginx-archive-keyring.gpg; \
    export GNUPGHOME="$(mktemp -d)"; \
    found=''; \
    for server in \
        hkp://keyserver.ubuntu.com:80 \
        pgp.mit.edu \
    ; do \
        echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
        gpg1 --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
    done; \
    test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
    gpg1 --export "$NGINX_GPGKEY" > "$NGINX_GPGKEY_PATH" ; \
    rm -rf "$GNUPGHOME"; \
    apt-get remove --purge --auto-remove -y gnupg1 && rm -rf /var/lib/apt/lists/* \
    && dpkgArch="$(dpkg --print-architecture)" \
    && nginxPackages=" \
        nginx=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-xslt=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-geoip=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-image-filter=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-njs=${NGINX_VERSION}+${NJS_VERSION}-${PKG_RELEASE} \
    " \
    && case "$dpkgArch" in \
        amd64|arm64) \
# arches officialy built by upstream
            echo "deb [signed-by=$NGINX_GPGKEY_PATH] https://nginx.org/packages/mainline/debian/ bullseye nginx" >> /etc/apt/sources.list.d/nginx.list \
            && apt-get update \
            ;; \
        *) \
# we're on an architecture upstream doesn't officially build for
# let's build binaries from the published source packages
            echo "deb-src [signed-by=$NGINX_GPGKEY_PATH] https://nginx.org/packages/mainline/debian/ bullseye nginx" >> /etc/apt/sources.list.d/nginx.list \
            \
# new directory for storing sources and .deb files
            && tempDir="$(mktemp -d)" \
            && chmod 777 "$tempDir" \
# (777 to ensure APT's "_apt" user can access it too)
            \
# save list of currently-installed packages so build dependencies can be cleanly removed later
            && savedAptMark="$(apt-mark showmanual)" \
            \
# build .deb files from upstream's source packages (which are verified by apt-get)
            && apt-get update \
            && apt-get build-dep -y $nginxPackages \
            && ( \
                cd "$tempDir" \
                && DEB_BUILD_OPTIONS="nocheck parallel=$(nproc)" \
                    apt-get source --compile $nginxPackages \
            ) \
# we don't remove APT lists here because they get re-downloaded and removed later
            \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
# (which is done after we install the built packages so we don't have to redownload any overlapping dependencies)
            && apt-mark showmanual | xargs apt-mark auto > /dev/null \
            && { [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; } \
            \
# create a temporary local APT repo to install from (so that dependency resolution can be handled by APT, as it should be)
            && ls -lAFh "$tempDir" \
            && ( cd "$tempDir" && dpkg-scanpackages . > Packages ) \
            && grep '^Package: ' "$tempDir/Packages" \
            && echo "deb [ trusted=yes ] file://$tempDir ./" > /etc/apt/sources.list.d/temp.list \
# work around the following APT issue by using "Acquire::GzipIndexes=false" (overriding "/etc/apt/apt.conf.d/docker-gzip-indexes")
#   Could not open file /var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages - open (13: Permission denied)
#   ...
#   E: Failed to fetch store:/var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages  Could not open file /var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages - open (13: Permission denied)
            && apt-get -o Acquire::GzipIndexes=false update \
            ;; \
    esac \
    \
    && apt-get install --no-install-recommends --no-install-suggests -y \
                        $nginxPackages \
                        gettext-base \
                        curl \
    && apt-get remove --purge --auto-remove -y && rm -rf /var/lib/apt/lists/* /etc/apt/sources.list.d/nginx.list \
    \
# if we have leftovers from building, let's purge them (including extra, unnecessary build deps)
    && if [ -n "$tempDir" ]; then \
        apt-get purge -y --auto-remove \
        && rm -rf "$tempDir" /etc/apt/sources.list.d/temp.list; \
    fi \
# forward request and error logs to docker log collector
    && ln -sf /dev/stdout /var/log/nginx/access.log \
    && ln -sf /dev/stderr /var/log/nginx/error.log \
# create a docker-entrypoint.d directory
    && mkdir /docker-entrypoint.d

# COPY docker-entrypoint.sh /
# COPY 10-listen-on-ipv6-by-default.sh /docker-entrypoint.d
# COPY 20-envsubst-on-templates.sh /docker-entrypoint.d
# COPY 30-tune-worker-processes.sh /docker-entrypoint.d
# ENTRYPOINT ["/docker-entrypoint.sh"]

EXPOSE 80
# expose these ports on the docker virtual network
# you still need to use -p or -P to open/forward these ports on host

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
# required: run this command when container is launched
# only one CMD allowed, so if there are multiple, last one wins
```

Now we have `Dockerfile` ready in the cwd. Let's build it.

```
docker image build -t <tag> <image-name-to-install>
```

Everytime docker runs, it builds with layer.

```
Step 1/6 : FROM debian:jessie
 ---> dfaslifjsdfs12
Step 2/6 : ENV NGINX_VERSION 1.11.10-1~jessie
 ---> Running in 3412ce412cd
 ---> 12v312kuhd12
```

As you can see every step is stored as cache so that when you reinstall to another container or update Dockerfile, it would no longer need to run the steps again which reduces the usage of CPU and unnecessary runs of the file.

`FROM` - image to pull from docker hub. (required)
`ENV` - Setting the envrionment variables
`RUN` - lets you to run commands. Each commands must be concatanated using `&&`
`EXPOSE` - By default, no TCP/UDP ports are open. This allows port from outside to container.
`CMD` - Required parameter that is the final command that runs everytime you start a container.
`WORKDIR` - Allows you to switch working directory.

Here's a better table for docker image

| Dockerfile Instruction | Explanation                                                                                                                                                                                                                                                                             |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| FROM                   | To specify the base image which can be pulled from a container registry( Docker hub, GCR, Quay, ECR, etc)                                                                                                                                                                               |
| RUN                    | Executes commands during the image build process.                                                                                                                                                                                                                                       |
| ENV                    | Sets environment variables inside the image. It will be available during build time as well as in a running container. If you want to set only build-time variables, use ARG instruction.                                                                                               |
| COPY                   | Copies local files and directories to the image. Syntax: COPY \<src-path\> \<destination-path\>                                                                                                                                                                                         |
| EXPOSE                 | Specifies the port to be exposed for the Docker container.                                                                                                                                                                                                                              |
| ADD                    | It is a more feature-rich version of the COPY instruction. It also allows copying from the URL that is the source and tar file auto-extraction into the image. However, usage of COPY command is recommended over ADD. If you want to download remote files, use curl or get using RUN. |
| WORKDIR                | Sets the current working directory. You can reuse this instruction in a Dockerfile to set a different working directory. If you set WORKDIR, instructions like RUN, CMD, ADD, COPY, or ENTRYPOINT gets executed in that directory.                                                      |
| VOLUME                 | It is used to create or mount the volume to the Docker container                                                                                                                                                                                                                        |
| USER                   | Sets the user name and UID when running the container. You can use this instruction to set a non-root user of the container.                                                                                                                                                            |
| LABEL                  | It is used to specify metadata information of Docker image                                                                                                                                                                                                                              |
| ARG                    | Is used to set build-time variables with key and value. the ARG variables will not be available when the container is running. If you want to persist a variable on a running container, use ENV.                                                                                       |
| CMD                    | It is used to execute a command in a running container. There can be only one CMD, if multiple CMD there then it only applies to the last one. It can be overridden from the Docker CLI.                                                                                                |
| ENTRYPOINT             | Specifies the commands that will execute when the Docker container starts. If you don’t specify any ENTRYPOINT, it defaults to /bin/sh -c. You can also override ENTRYPOINT using the --entrypoint flag using CLI. Please refer CMD vs ENTRYPOINT for more information.                 |
|                        |
