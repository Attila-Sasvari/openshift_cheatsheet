# OpenShift Cheatsheet

## Table of Contents

* [Authentication](#authentication)
  - [Podman](#podman-login)
  - [OpenShift](#openshift-login)
* [Container Tools](#container-tools)
  - [Podman commands](#podman-commands)
  - [Skopeo commands](#skopeo-commands)
* [Images](#images)
  - [Import Image](#import-image)
  - [Internal Image Registry](#internal-image-registry)
  - [S2I - Source to Image](#s2i---source-to-image)
* [Templates](#templates)
* [New App](#new-app)
* [Config Map](#config-map)
* [Secret](#secret)
* [Build Hooks](#build-hooks)
* [Copy Files](#copy-files)
* [Expose Service](#expose-service)
* [Routes](#routes)
* [Set Env](#set-env)
* [Run Command Inside a Pod](#run-command-inside-a-pod)
* [Rollout](#rollout)
* [Triggers](#triggers)
* [Probes](#probes)
* [Accessing Data](#accessing-data)

## Authentication

### Podman login

```bash
# login with podman
podman login quay.io -u ${USER}

# logout with podman
podman logout quay.io
```

After successful authentication, Podman stores an access token in the `/run/user/UID/containers/auth.json` file. The /run/user/UID path prefix is not fixed and comes from the `XDG_RUNTIME_DIR` environment variable.

### OpenShift login

```bash
# login with oc
oc login -u ${USER} -p ${PASSWORD} ${OCP4_MASTER_API}

# insecure login
oc login --username=${USER} --insecure-skip-tls-verify --server=${OCP4_MASTER_API}
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



## Container Tools

### Podman Commands

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

### Skopeo Commands

```bash
# options: oci | docker | containers-storage
# check an image in a registry with credentials
skopeo inspect --creds user:password \
  docker://registry.redhat.io/rhscl/postgresql-96-rhel7

# copy from local image to an external without TLS verification
skopeo copy --dest-tls-verify=false \
  containers-storage:localhost/myimage \
  docker://registry.example.com/myorg/myimage

# copy between registries with credentials
skopeo copy --src-creds=user:password \
  --dest-creds=user1:password \
  docker://srcregistry.domain.com/org1/private \
  docker://dstegistry.domain2.com/org2/private

# copy from OCI-formatted folder
skopeo copy oci:/home/user/myimage \
  docker://registry.example.com/myorg/myimage

# delete external image
skopeo delete \
  docker://quay.io/${RHT_OCP4_QUAY_USER}/ubi-sleep:1.0
```

## Images

### Import Image

```bash
# import image with all tags
oc import-image <image name> \
  --from=<registryserver:port>/<repo>/<imagename> --all --confirm

# import specific tag, insecure mode, cache image layers in the internal registry
oc import-image <image name[:tag]> \
  --from=<registryserver:port>/<repo>/<imagename[:tag]> \
  --confirm --insecure
  --reference-policy local

# update image after imported
oc import-image <image name>
```

### Internal Image Registry

```bash
# get the OpenShift token as a password
TOKEN=$(oc whoami -t)

# find route of the internal image registry
oc get route -n openshift-image-registry

# login to the registry
podman login -u ${USER} -p ${TOKEN} \
  default-route-openshift-image-registry.domain.example.com

# inspect an image with credential
skopeo inspect --creds=${USER}:${TOKEN} \
  docker://default-route-openshift-image-registry.domain.example.com/myproj/myapp

# inspect an image without TLS certification
skopeo inspect --tls-verify=false \
  docker://default-route-openshift-image-registry.domain.example.com/myproj/myapp

# add a user a local role to be able to pull image
# roles: registry-viewer | registry-editor | system:image-pusher | system:image-puller
oc policy add-role-to-user system:image-puller \
  user_name -n project_name

# how to add cluster role if needed
oc adm policy add-cluster-role-to-user <role> <username>

# details of an image stream in a given namespace
oc describe is php -n openshift

# allow containers run with root user inside openshift
oc adm policy add-scc-to-user anyuid -z default
```

### S2I - Source to Image

## Templates

```bash
# create new app from template file, specifying parameter values
oc new-app --file mytemplate.yaml \
  -p PARAM1=value1 \
  -p PARAM2=value2

# applies values to a template and stores the results in a local file
oc process -f mytemplate.yaml -p PARAM1=value1 \
  -p PARAM2=value2 > myresourcelist.yaml

# create based on the previous output file
oc create -f myresourcelist.yaml

# combine the previous two steps
oc process -f mytemplate.yaml -p PARAM1=value1 \
  -p PARAM2=value2 | oc create -f -

# list of available templates generally
oc get templates -n openshift

# create template file so it can be edited
oc export is,bc,dc,svc,route --as-template > mytemplate.yml
oc export all --as-template=<template_name>

# get the list of parameters of a template file
oc process -f mytemplate.yaml --parameters

# get the list of parameters of a template
oc process --parameters=true -n openshift container-name

# get the details of a template
oc describe template some_template -n ${NAMESPACE}

# create template to json
oc new-app -o json openshift/container > container.json

# validate a template
oc create --dry-run --validate -f openshift/template/some-buildconfig.yaml
```

Red Hat recommends using the oc new-app command rather than the oc process command.

## New App

```bash
# create new app as deployment from source, using build-env, context dir, branch
oc new-app --name someapp \
  --build-env npm_config_registry=\
  http://${NPM_SERVER}/repository/nodejs \
  nodejs:12~https://github.com/${GITHUB_USER}/some-repo#some-branch \
  --context-dir some-subdir

# create new app as deployment config
oc new-app --as-deployment-config --name someapp \
  --build-env MAVEN_MIRROR_URL=http://${NEXUS_SERVER}/repository/java \
  -i redhat-openjdk18-openshift:1.5 --context-dir somedir \
  https://github.com/${GITHUB_USER}/some-repo#some-branch

# create new app from remote image
oc new-app --name someapp \
  --docker-image quay.io/${QUAY_USER}/ubi-something:1.0

# create new app from template
oc new-app --template=mysql-ephemeral \
  --param=MYSQL_USER=mysqluser,MYSQL_PASSWORD=p5ssw0rd,MYSQL_DATABASE=mydb,DATABASE_SERVICE_NAME=database

# another way to define parameters
oc new-app \
  mysql -e MYSQL_USER=user -e MYSQL_PASSWORD=pass \
  -e MYSQL_DATABASE=testdb -l db=mysql

# --strategy options: docker | source | pipeline
# --dry-run=true is the result of operation without performing it
# --code is like a GitHub repo

# delete everything of an app by specifying a label
oc delete all -l db=mysql
```


## Config Map

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

## Secret

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



## Service Account

```bash
# create new service account
oc create serviceaccount myserviceaccount

# how to update a deployment config (or use oc edit)
oc patch dc/demo-app --patch \
  '{"spec":{"template":{"spec":{"serviceAccountName": "myserviceaccount"}}}}'

# anyuid SCC to run using a fixed userid in the container
oc adm policy add-scc-to-user anyuid -z myserviceaccount

# allows a service account to pull the image layers that the
# image stream cached in the internal registry
oc policy add-role-to-group system:image-puller \
  system:serviceaccounts:myapp
```

## Build Hooks

```bash
oc set build-hook bc/name --post-commit \
  --command -- bundle exec rake test --verbose

oc set build-hook bc/name --post-commit \
  --script="curl http://api.com/user/${USER}"

oc set env bc/hook DEVELOPER="Your Name"

oc set env bc/hook --list

oc start-build bc/hook -F
```

## Copy Files

```bash
# Copy file from the container, and back
oc cp <container_name>:/var/log/some_path /tmp/local_path

oc cp /tmp/local_path <container_name>:/var/log/some_path

# the same with rsync
oc rsync /home/user/source <container_name>:/src

oc rsync <container_name>:/src /home/user/source
```

## Expose Service

```bash

oc expose svc/greet

# to use an external service inside the cluster
oc create service externalname myservice \
  --external-name myhost.example.com
```


## Routes

```bash
# find route of the console
oc get route -n openshift-console

# find route of the internal image registry
oc get route -n openshift-image-registry
```

## Set Env

```bash
oc env dc/registry STORAGE=/data
oc env dc/registry --overwrite STORAGE=/opt

# remove env variables
oc env dc/d1 ENV1- ENV2-
oc env rc --all ENV-
```


## Run Command Inside a Pod

```bash
# one time command
oc rsh container_name-1-id ps ax

# interactive shell
oc rsh -t container_name-1-id

# run a shell command in a pod
oc rsh container_name-1-id bash -c \
  'echo > /dev/tcp/$DATABASE_SERVICE_NAME/3306 && echo OK || echo FAIL'

# get env variables of a pod
oc rsh container_name-1-id env
```

## Rollout

```bash
# force a new deployment to test changes
oc rollout latest dc/mysql

# get the rollout history
oc rollout history dc/name

# get the specifics of a given rollout
oc rollout history dc/name --revision=1

# the names of the following are self-explanatory
oc rollout cancel dc/name
oc rollout retry dc/name
oc rollout pause dc <dc name>
oc rollout resume dc <dc name>
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

# disable triggers, e.g., during multiple changes
oc set triggers dc/mysql --from-config --remove

# re-enable trigger after the changes
oc set triggers dc/name --auto
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

## Accessing Data

```bash
REGISTRY=$(oc get routes -n default docker-registry -o jsonpath='{.spec.host}')

http://$(oc get route nexus3 --template='{{ .spec.host }}')

oc patch dc/mysql --patch \
'{"spec":{"strategy":{"type":"Recreate"}}}'

```
