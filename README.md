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
* [Persistent Volume Claims](#persistent-volume-claims)
* [Set Env](#set-env)
* [Run Command Inside a Pod](#run-command-inside-a-pod)
* [Scale Deployment](#scale-deployment)
* [Rollout](#rollout)
* [Triggers](#triggers)
* [Probes](#probes)
* [Events](#events)
* [OpenShift Administration](#openshift-administration)
* [Accessing Data](#accessing-data)

## Authentication

### Podman login

```bash
# login with podman
podman login quay.io -u ${USER}

# logout with podman
podman logout quay.io
```

After successful authentication, Podman stores an access token in the `/run/user/UID/containers/auth.json` file. The `/run/user/UID` path prefix is not fixed and comes from the `XDG_RUNTIME_DIR` environment variable.

### OpenShift login

```bash
# login with oc
oc login -u ${USER} -p ${PASSWORD} ${OCP4_MASTER_API}

# insecure login
oc login --username=${USER} --insecure-skip-tls-verify --server=${OCP4_MASTER_API}
```

It is possible to create a secret based on the podman login information, then link to internal service accounts.

```bash
oc create secret generic secret_name \
  --from-file .dockerconfigjson=${XDG_RUNTIME_DIR}/containers/auth.json \
  --type kubernetes.io/dockerconfigjson

# to pull image
oc secrets link default secret_name --for pull

# to build image
oc secrets link builder secret_name
```

The OpenShift installer creates an auth directory containing the `kubeconfig` and `kubeadmin` password files. Run the `oc login command` to connect to the cluster with the kubeadmin user.

The password of the kubeadmin user is in the kubeadmin-password file.

```bash
oc login -u kubeadmin -p ${PASSWORD} ${OCP4_MASTER_API}:6443
```


## Container Tools

### Podman Commands

```bash
# login to an image registry
podman login quay.io -u ${USER}

# see locally available images
podman images

# get formatted list in a table format
podman images \
  --format "table {{.ID}} {{.Repository}} {{.Tag}}"

# search for a specific image in a registry
podman search quay.io/ubi-sleep

# pull an image
podman pull rhel

# create an image
podman run ubi8/ubi:8.3 echo 'Hello world!'

# build an image from a containerfile
podman build --layers=false -t do288-apache ./container-build

# run a container in the background
podman run -d --name some-container \
  -e SOME_PARAM=value \
  quay.io/${USER}/ubi-sleep:1.0

# example to port forwarding
podman run -d --name apache1 -p 8080:80 \
  registry.redhat.io/rhel8/httpd-24

# port forward to a container
oc port-forward mysql-openshift-1-glqrp 3306:3306

# list the ports of the last used container
podman port -l

# open an interactive shell to a container
podman exec -it mysql-basic /bin/bash

# run a command in the container using the last one
podman exec -l cat /etc/hostname

# get the list of running containers
podman ps
podman ps --format "{{.ID}} {{.Image}} {{.Names}}"
podman ps --format="table {{.ID}} {{.Names}} {{.Image}} {{.Status}}"

# create tar file locally from an image
podman save \
  -o mysql.tar registry.redhat.io/rhel8/mysql-80

# load back an image from a tar file
podman load -i mysql.tar

# check the modifications of an image
podman diff mysql-basic

# commit the changes
podman commit mysql-basic mysql-custom

# tag a local image
podman tag do288-apache quay.io/${USER}/do288-apache

# push a local image to a registry
podman push quay.io/${USER}/do288-apache

# see the logs of a container
podman logs my-httpd-container

# stop a container
podman stop my-httpd-container

# stop and start a container
podman restart my-httpd-container

# remove a container
podman rm sleep

# remove image even if there is a corresponding container
podman rmi -a --force

# if a volume to be added
mkdir /home/student/dbfiles
# provides a session to execute commands within the same user namespace
# as the process running inside the container
podman unshare chown -R 27:27 /home/student/dbfiles
sudo semanage fcontext -a -t container_file_t
  '/home/student/dbfiles(/.*)?'
sudo restorecon -Rv /home/student/dbfiles

# mount a volume
podman run -v /home/student/dbfiles:/var/lib/mysql rhmap47/mysql
```

Registries can be defined for Podman in `/etc/containers/registries.conf`, which has content like:

```bash
[registries.search]
registries = ["registry.access.redhat.com", "quay.io"]

[registries.insecure]
registries = ['localhost:5000']
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

# available image streams
oc get is -n openshift

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

```bash
# check where the s2i scripts are, e.g., "io.openshift.s2i.scripts-url": "image:///usr/libexec/s2i"
skopeo inspect \
  docker://myregistry.example.com/rhscl/php-73-rhel7 \
  | grep io.openshift.s2i.scripts-url

# an assemble or run scripts can be written is .s2i/bin folder
```

The assemble script can be referenced in .sh script as `/usr/libexec/s2i/assemble`.
The run script must use exec to invoke it: `exec /usr/libexec/s2i/run`.

The save-artifacts script ensures that dependent artifacts (libraries and components required for the application) are saved for future builds.
The save-artifacts script output should only include the tar stream output, and nothing else. You should redirect output of other commands in the script to /dev/null.

```bash
# install package
sudo yum install source-to-image

# check version of s2i
s2i version

# create s2i directory
s2i create <image_name> <directory>

# creates directory like this
directory
├── Dockerfile
├── Makefile
├── README.md
├── s2i
│ └── bin
│ ├── assemble
│ ├── run
│ ├── save-artifacts
│ └── usage
└── test
 ├── run
 └── test-app
 └── index.html

# the Dockerfile is to have a COPY instruction to copy
# ./,s2i/bin/ to /user/lebexex/s2i or whatever skope inspect says

# also add LABELs, like
LABEL io.k8s.description="A basic Apache HTTP Server S2I builder image" \
  io.k8s.display-name="Apache HTTP Server S2I builder image for DO288" \
  io.openshift.expose-services="8080:http" \
  io.openshift.s2i.scripts-url="image:///usr/libexec/s2i" \
  io.openshift.tags="builder, httpd, httpd24"

# once updated, build the image.
# format docker only needed when ONBUILD instruction is used in Dockerfile
podman build -t builder_image --format docker .

# then build an app container with s2i
s2i build <src> <builder_image> <tag_name> --as-docker-file /path/to/Dockerfile
# Dockerfile should include the s2i scripts

# build locally from the Dockerfile
podman build -t nginx-test /path/to/Dockerfile
# run with a different random user
podman run -u 1234 -d -p 8080:8080 nginx-test

# copy to the image registry
skopeo copy \
  containers-storage:localhost/s2i-do288-httpd \
  docker://quay.io/${QUAY_USER}/s2i-do288-httpd

# then create secret from .dockerconfigjson
# then oc secrets link builder <registry>
# then import-image
# verify with oc get is
# finally create new app
```

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
oc get templates -n openshift -o yaml

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

# get more details about pods, like IP
oc get pods -o wide
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

# set environment var from secret with prefix
oc set env deployment/mydcname \
  --from secret/mysecret --prefix MYSQL_

# set secret values as a volume
oc set volume deployment/mydcname --add \
  --type secret --mount-path /path/to/mount/volume \
  --name myvol --secret-name mysecret

# create ssh secret
oc create secret generic ssh-keys \
  --from-file id_rsa=/path-to/id_rsa \
  --from-file id_rsa.pub=/path-to/id_rsa.pub

# create TLS secret
oc create secret tls secret-tls \
  --cert /path-to-certificate --key /path-to-key

# update a secret
oc extract secret/htpasswd-ppklq -n openshift-config \
  --to /tmp/ --confirm
# then make the update
# finally set again the secret
oc set data secret/htpasswd-ppklq -n openshift-config \
  --from-file /tmp/htpasswd
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

The post-commit build hook runs commands in a temporary container before pushing the new container image generated by the build to the registry.

```bash
# add a command after --
oc set build-hook bc/name --post-commit \
  --command -- bundle exec rake test --verbose

oc set build-hook bc/name --post-commit \
  --script="curl http://api.com/user/${USER}"

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

# some further parameters to know
# --hostname
# --port: the port that the resource should serve on
# --target-port: on the container
# --name
# --protocol
# --type, e.g. LoadBalancer, ClusterIP
```


## Routes

```bash
# find route of the console
oc get route -n openshift-console

# find route of the internal image registry
oc get route -n openshift-image-registry

# by default, router is deployed in openshift-ingress project
oc get pod --all-namespaces -l app=router
```

## Persistent Volume Claims

```bash
# see available storage classes
oc get storageclass

oc set volumes deployment/example-application \
  --add --name example-storage --type pvc --claim-class nfs-storage \
  --claim-mode rwo --claim-size 15Gi --mount-path /var/lib/example-app \
  --claim-name example-storage

oc set volumes \
  deployment/postgresql-persistent2 \
  --add --name postgresql-storage --type pvc \
  --claim-name postgresql-storage --mount-path /var/lib/pgsql

oc get pvc

oc delete pvc/example-pvc-storage
```

## Set Env

```bash
oc set env dc/registry STORAGE=/data
oc set env dc/registry --overwrite STORAGE=/opt

# remove env variables
oc set env dc/d1 ENV1- ENV2-
oc set env rc --all ENV-
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

## Scale Deployment

```bash
# scale replices to a specific number
oc scale --replicas 5 deployment/scale

# create a horizontal pod autoscaler
oc autoscale dc/hello --min 1 --max 10 --cpu-percent 80

# get information about horizontal pod autoscaler resources in the current project
oc get hpa
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

## Events

```bash
oc get events
```

## OpenShift Administration

```bash
oc get nodes
oc describe node my-node-name

oc adm node-logs -u crio my-node-name
oc adm node-logs my-node-name


# displays the current CPU and memory usage of each node
oc adm top nodes

# retrieve the cluster version and its details
oc get clusterversion
oc describe clusterversion

# retrieve the list of all cluster operators with their status
oc get clusteroperators
```

Debug

```bash
# debug a node
oc debug node/my-node-name
...output omitted...
sh-4.2# chroot /host
sh-4.2# systemctl is-active kubelet
active
sh-4.2# systemctl status kubelet
sh-4.2# systemctl status cri-o
# get low-level information about all local containers running on the node
sh-4.2# crictl ps

# debug a deployment
oc debug deployment/my-deployment-name --as-root
```

Select loglevel

```bash
oc get pod --loglevel 10
```

## Accessing Data

Here are some random examples.

```bash

oc get route -o jsonpath='{..spec.host}{"\n"}'

REGISTRY=$(oc get routes -n default docker-registry -o jsonpath='{.spec.host}')

http://$(oc get route nexus3 --template='{{ .spec.host }}')

oc patch dc/mysql --patch \
'{"spec":{"strategy":{"type":"Recreate"}}}'

```
