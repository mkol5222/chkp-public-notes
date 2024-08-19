# Docker API and bash scripts

```shell
# assume Linux with Docker installed
docker info

# current user is member of docker group - able to access docker socket without sudo
ls -la /var/run/docker.sock
id

# Docker API is exposed as a Unix socket
# it is HTTP API and curl (HTTP command line client) can be used to access it
curl --unix-socket /var/run/docker.sock http://localhost/info 
# even better with jq (JSON processor)
curl --unix-socket /var/run/docker.sock http://localhost/info | jq

# lets list all container images
curl --unix-socket /var/run/docker.sock http://localhost/images/json | jq
# human readable name+tag and id only
curl --unix-socket /var/run/docker.sock http://localhost/images/json | jq -r '.[] | [.RepoTags[], .Id] | @csv'
# compare with
docker images

# lets pull an image
curl --unix-socket /var/run/docker.sock -X POST http://localhost/images/create?fromImage=alpine:3.12
# compare with
docker image rm  alpine:3.12; docker pull alpine:3.12

# lets save an image to tar file
curl --unix-socket /var/run/docker.sock http://localhost/images/alpine:3.12/get > alpine-3.12.tar
file alpine-3.12.tar
tar tvf alpine-3.12.tar
# compare with
docker save alpine:3.12 -o alpine-3.12.tar
file alpine-3.12.tar
tar tvf alpine-3.12.tar

# such tar file may be subject to inspection with Check Point CloudGuard ShiftLeft scanner

```
