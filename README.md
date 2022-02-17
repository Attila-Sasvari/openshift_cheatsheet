# OpenShift Cheatsheet

## Authentication

### Podman

```bash
# login with podman
podman login quay.io -u ${USER}

# logout with podman
podman logout quay.io
```

After successful authentication, Podman stores an access token in the `/run/user/UID/containers/auth.json` file. The /run/user/UID path prefix is not fixed and comes from the `XDG_RUNTIME_DIR` environment variable.

### OpenShift

```bash
# login with oc
oc login -u ${USER} -p ${PASSWORD} ${OCP4_MASTER_API}

# insecure login
oc login --username=tuelho --insecure-skip-tls-verify --server=https://master00-${guid}.oslab.opentlc.com:8443
```

It is possible to create a secret based on the podman login information.

```bash
oc create secret generic secret_name \
  --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json \
  --type kubernetes.io/dockerconfigjson
```

Then link to internal service accounts.

```bash
# to pull image
oc secrets link default secret_name --for pull

# to build image
oc secrets link builder secret_name
```

## Podman Commands

```bash
# login to an image registry
podman login quay.io -u ${USER}

# see locally available images
podman images

# search for a specific image in a registry
podman search quay.io/ubi-sleep

# get the list of local containers
podman ps

# build an image from a containerfile
podman build --layers=false -t do288-apache ./container-build

# tag a local image
podman tag do288-apache quay.io/${USER}/do288-apache

# push a local image to a registry
podman push quay.io/${USER}/do288-apache

# run a container in the background
podman run -d --name sleep quay.io/${USER}/ubi-sleep:1.0

# see the logs of a container
podman logs sleep

# stop a container
podman stop sleep

# remove a container
podman rm sleep

# remove image even if there is a corresponding container
podman rmi -a --force
```

## Skopeo Commands

```bash
# options: oci | docker | containers-storage
# check an image in a registry with credentials
skopeo inspect --creds user:password \
  docker://registry.redhat.io/rhscl/postgresql-96-rhel7

# copy from local image to an external without TLS verification
skopeo copy --dest-tls-verify=false \
  containers-storage:myimage \
  docker://registry.example.com/myorg/myimage

# copy between registries with credentials
skopeo copy --src-creds=user:password \
  --dest-creds=user1:password \
  docker://srcregistry.domain.com/org1/private \
  docker://dstegistry.domain2.com/org2/private

# copy from OCI-formatted folder
skopeo copy oci:myimage \
  docker://registry.example.com/myorg/myimage

# delete external image
skopeo delete \
  docker://quay.io/${RHT_OCP4_QUAY_USER}/ubi-sleep:1.0
```

## Routes

```bash
# find route of the console
oc get route -n openshift-console

# find route of the internal image registry
oc get route -n openshift-image-registry
```

## Config map

```bash
# create configmap from key-value pairs
oc create configmap config_map_name \
  --from-literal key1=value1 \
  --from-literal key2=value2

# create configmap from file
oc create configmap config_map_name \
  --from-file /home/demo/conf.txt

# get details of the configmap
oc get cm myconf
# gives back the value of a parameter
oc describe cm/myconf

# get details of the configmap as a json
oc get configmap/myconf -o json

# edit configmap
oc edit configmap/myconf

# delete configmap
oc delete configmap/myconf

# update configmap
oc patch configmap/myconf --patch '{"data":{"key1":"newvalue1"}}'

# set environment var from configmap
oc set env deployment/mydcname \
  --from configmap/myconf

# set configmap values as a volume
oc set volume deployment/mydcname --add \
  -t configmap -m /path/to/mount/volume \
  --name myvol --configmap-name myconf
```

## Secrets

```bash
# create configmap from key-value pairs
oc create secret generic secret_name \
  --from-literal username=user1 \
  --from-literal password=mypa55w0rd

# create configmap from file
oc create secret generic secret_name \
  --from-file /home/demo/mysecret.txt

# edit secret
oc edit secret/mysecret
# the new value must be base63 encoded
# encode string first:
echo 'newpassword' | base64

# edit or path with the new value
oc patch secret/mysecret --patch \
  '{"data":{"password":"bmV3cGFzc3dvcmQK"}}'

# two ways to create secret for login
# option 1
oc create secret docker-registry registrycreds \
  --docker-server registry.example.com \
  --docker-username youruser \
  --docker-password yourpassword

# option 2
oc create secret generic registrycreds \
  --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json \
  --type kubernetes.io/dockerconfigjson

# then link the secret
oc secrets link default registrycreds --for pull

# to access an S2I builder image
oc secrets link builder registrycreds

# set environment var from secret
oc set env deployment/mydcname \
  --from secret/mysecret

# set secret values as a volume
oc set volume deployment/mydcname --add \
  -t secret -m /path/to/mount/volume \
  --name myvol --secret-name mysecret

# create ssh secret
oc create secret generic sshsecret \
  --from-file=ssh-privatekey=$HOME/.ssh/id_rsa
```

## json path examples

```bash
REGISTRY=$(oc get routes -n default docker-registry -o jsonpath='{.spec.host}')

http://$(oc get route nexus3 --template='{{ .spec.host }}')
```

## Internal image repository

```bash
# get the OpenShift token as a password
TOKEN=$(oc whoami -t)

# find route of the internal image registry
oc get route -n openshift-image-registry

# login to the registry
podman login -u myuser -p ${TOKEN} \
  default-route-openshift-image-registry.domain.example.com

# inspect an image with credential
skopeo inspect --creds=myuser:${TOKEN} \
  docker://default-route-openshift-image-registry.domain.example.com/myproj/myapp

# inspect an image without TLS certification
skopeo inspect --tls-verify=false \
  docker://default-route-openshift-image-registry.domain.example.com/myproj/myapp

# add a user a local role to be able to pull image
# roles: registry-viewer | registry-editor | system:image-pusher | system:image-puller
oc policy add-role-to-user system:image-puller \
  user_name -n project_name

# cluster role
oc adm policy add-cluster-role-to-user <role> <username>

# details of an image stream in a given namespace
oc describe is php -n openshift

# allow containers run with root user inside openshift
oc adm policy add-scc-to-user anyuid -z default

```

## Copy file between the container and local machine

```bash
# Copy file from the container
oc cp container_name:/var/log/some_path /tmp/local_path

oc rsync /home/user/source devpod1234:/src

oc rsync devpod1234:/src /home/user/source
```

## Run command inside a pod

```bash
# one time command
oc rsh container_name-1-zvjhb ps ax

# interactive shell
oc rsh -t container_name-1-zvjhb

oc rsh quotesapi-7d76ff58f8-6j2gx bash -c \
  'echo > /dev/tcp/$DATABASE_SERVICE_NAME/3306 && echo OK || echo FAIL'
```

## Templates

```bash
oc new-app --file mytemplate.yaml -p PARAM1=value1 \
  -p PARAM2=value2

oc process -f mytemplate.yaml -p PARAM1=value1 \
  -p PARAM2=value2 > myresourcelist.yaml

# list of available templates generally
oc get templates -n openshift

oc create -f some_template.json
oc create -f myresourcelist.yaml

oc process -f mytemplate.yaml --parameters

oc describe template some_template -n ${USER}-common
# e.g., describe svc gives back the parameters, the endpoint IP

# list parameters
oc process --parameters=true -n openshift mysql-persistent

# create template to json
oc new-app -o json openshift/hello-openshift > hello.json

# validate a template
oc create --dry-run --validate -f openshift/template/tomcat6-docker-buildconfig.yaml
```

Red Hat recommends using the oc new-app command rather than the oc process command.


## New app

```bash
oc new-app --name greet \
  --build-env npm_config_registry=\
  http://${RHT_OCP4_NEXUS_SERVER}/repository/nodejs \
  nodejs:12~https://github.com/${RHT_OCP4_GITHUB_USER}/DO288-apps#source-build \
  --context-dir nodejs-helloworld

oc new-app --as-deployment-config --name quip \
  --build-env MAVEN_MIRROR_URL=http://${RHT_OCP4_NEXUS_SERVER}/repository/java \
  -i redhat-openjdk18-openshift:1.5 --context-dir quip \
  https://github.com/${RHT_OCP4_GITHUB_USER}/DO288-apps#app-deploy


oc new-app --name sleep \
  --docker-image quay.io/${RHT_OCP4_QUAY_USER}/ubi-sleep:1.0

oc new-app --template=mysql-ephemeral \
  --param=MYSQL_USER=mysqluser,MYSQL_PASSWORD=redhat,MYSQL_DATABASE=mydb,DATABASE_SERVICE_NAME=database
```

## Expose Service

```bash
oc create service externalname myservice \
  --external-name myhost.example.com
```

## Triggers

```bash
# set triiger to build config
oc set triggers bc/name --from-image=project/image:tag

# to remove a trigger
oc set triggers bc/name --from-image=project/image:tag --remove

# other values: --from-bitbucket | --from-github
oc set triggers bc/name --from-gitlab

oc set triggers bc/name --from-gitlab --remove
```

## Build hook

```bash
oc set build-hook bc/name --post-commit \
  --command -- bundle exec rake test --verbose

oc set build-hook bc/name --post-commit \
  --script="curl http://api.com/user/${USER}"

oc set env bc/hook DEVELOPER="Your Name"

oc set env bc/hook --list

oc start-build bc/hook -F
```

## Set Env

```bash
oc env dc/registry STORAGE=/data
oc env dc/registry --overwrite STORAGE=/opt

# remove env variables
oc env dc/d1 ENV1- ENV2-
oc env rc --all ENV-
```

## Probes

Readiness probes determine whether or not a container is ready to serve requests.

Liveness probes determine whether or not an application running in a container is in a
healthy state.

```bash
# works with deployment
oc set probe deployment probes --liveness \
  --get-url=http://:8080/healthz --period=20 \
  --success-threshold=1 --failure-threshold=3

# it can be either readiness or liveness
oc set probe deployment probes --readiness \
  --get-url=http://:8080/ready \
  --initial-delay-seconds=2 --timeout-seconds=2

# and works with deployment config too
oc set probe dc/quip \
  --liveness --readiness --get-url=http://:8080/ready \
  --initial-delay-seconds=30 --timeout-seconds=2

# not just get-url, but: open-tcp |
oc set probe deployment myapp --liveness \
  --open-tcp=3306 --period=20 \
  --timeout-seconds=1

# Clear both readiness and liveness probes off all containers
oc set probe dc/registry --remove --readiness --liveness

# Set an exec action as a liveness probe to run 'echo ok'
oc set probe dc/registry --liveness -- echo ok
```

## Rollout

```bash
oc rollout latest dc/mysql

oc rollout pause dc <dc name>

oc rollout resume dc <dc name>
```

## Export

```bash
oc export is,bc,dc,svc,route --as-template > mytemplate.yml

oc export all --as-template=<template_name>
```

## Import image

```bash
oc import-image <image name> --from=docker.io/<imagerepo>/<imagename> --all --confirm

```
