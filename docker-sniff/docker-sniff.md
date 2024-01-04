# Inspect traffic between Docker command and Docker daemon

## Introduction

This article will show you how to inspect the traffic between the Docker command and the Docker daemon.

## Prerequisites

* Ubuntu Linux with Docker installed

## How Docker command talks to the daemon

Docker daemon exposes a REST API. Docker command talks to the daemon via this REST API. The REST API is exposed via a Unix socket. The default location of the Unix socket is `/var/run/docker.sock`.

```shell
# look for the docker.sock file representing the Unix socket
ls -la /var/run/docker.sock

# by default DOCKER_HOST is not set, assuming the default value of /var/run/docker.sock
echo $DOCKER_HOST

# list docker images
docker image ls

# set DOCKER_HOST to the default value of /var/run/docker.sock
export DOCKER_HOST=unix:///var/run/docker.sock

# list docker images - it still works
docker image ls

# DOCKER_HOST can be set to a TCP address - e.g. Docker socket from server forwarded to local TCP port
# but will will skip it for now
# export DOCKER_HOST=tcp://localhost:62375
```

## Tool to inspect Unix socket traffic

We are looking for tool able to inspect Unix socket traffic. The tool should be able to inspect the traffic between the Docker command and the Docker daemon. 
It can be done with `socat` tool. `socat` is a command line based utility that establishes two bidirectional byte streams and transfers data between them. It is kind of netcat on steroids.

So we will listen on new Unix socket and forward all traffic to the original Unix socket. We will use `socat` to do that.
Plus we will copy the traffic to the standard output so we can see what is going on.

```shell
# install socat on Ubuntu
sudo apt update; sudo apt install socat -y

# syntax of socat command and supported streams
socat -h

# listen on new Unix socket and forward all traffic to the original Unix socket
# copy the traffic to the standard output so we can see what is going on
socat -v UNIX-LISTEN:/tmp/docker.sock,mode=777,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock
```

## Inspect traffic between Docker command and Docker daemon

Now we are ready to open one more terminal and try to list docker images. We should see the traffic between the Docker command and the Docker daemon.

```shell
# this would be done through the original Unix socket
docker image ls
# so no output shown by socat

# we need to update DOCKER_HOST to the new Unix socket
export DOCKER_HOST=unix:///tmp/docker.sock

# this would be done through the new Unix socket
docker image ls
# so we can see the output shown by socat
```

Look at the output of `socat` command. It shows the traffic between the Docker command and the Docker daemon.

## Make your own requests to the Docker daemon

Once we know thare is text based REST protocol between the Docker command and the Docker daemon we can make our own requests to the Docker daemon. E.g. using `curl` command.

```shell
# list images without use of Docker command
curl -vvv --unix-socket /var/run/docker.sock http:/v1.24/images/json

# if there were no images the output, pull some
docker pull hello-world

# use jq to format the output
curl -vvv --unix-socket /var/run/docker.sock http:/v1.24/images/json | jq

# show less
curl -s --unix-socket /var/run/docker.sock http:/v1.24/images/json | jq '.[] | {Id: .Id, RepoTags: .RepoTags[0]}'
```

## Conclusion

We have learned how to inspect the traffic between the Docker command and the Docker daemon. We have learned how to make our own requests to the Docker daemon.

## References

Inspiration from article [Inspecting docker activity with socat](https://developers.redhat.com/blog/2015/02/25/inspecting-docker-activity-with-socat).

