# Setting up a local Maximo Manage development environment

Informal scripts to automate the setup of podman container running local Maximo Manage (using OpenShift backend). Follow this guide for official reference: https://www.ibm.com/docs/en/mas-cd/maximo-manage/continuous-delivery?topic=administering-setting-up-local-maximo-manage-development-environment

## Prerequisite

1. You will need podman installed locally. 
2. This script assumes you have OpenShift registry setup for when you deployed Manage. The script will check if you may have forgotton to open the Route to the registry.
3. This script uses `kubeadmin` user. Adjust/change according to your cluster.
4. This script requires a present-working-directory (I use `mas8`) where you will create the below mentioned `Dockerfile` (for Podman, it is `Containerfile`). Place this file in `\mas8\manage-dev\manage-developer` directory.
5. You will need at least 24GB RAM on your local computer. You might encounter issues running the container if you do not have enough memory.

### NOTE

1. This solution uses your OpenShift where you have already installed Manage with a database using multiple server bundles.  
2. The **local Maximo Manage development environment** connects to your database. 
3. Replace the IP address and credentials to match yours. 
4. Do not connect to your production database for this solution.
5. Follow the above linked official documentation. For the script shared below, I cannot take responsibility for any changes or issues that you may encounter and not be able to resolve independently.

`local-manage.sh`

```shell
#!/usr/bin/bash
##
function confirm_oclogin() { oc whoami &>> /dev/null
        if [[ $? == "0" ]]; then echo "Logged in to OCP"; else echo "You are not logged in to OCP"; fi }
##
function confirm_registry() { oc get route default-route -n openshift-image-registry &>> /dev/null
        if [[ $? == "0" ]]; then export REGISTRY=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}') && echo $REGISTRY; else oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge && export REGISTRY=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}') && echo $REGISTRY; fi }
##
confirm_oclogin
confirm_registry
##
podman login -u kubeadmin -p $(oc whoami -t) --tls-verify=false $REGISTRY
##
MAS_INSTANCE_ID=$(oc get suite --all-namespaces --ignore-not-found -o jsonpath='{.items[*].metadata.name}') ; echo $MAS_INSTANCE_ID
##
oc project mas-${MAS_INSTANCE_ID}-manage
##
MAS_WORKSPACE_ID=$(oc -n mas-masdemo-core get Workspace -o jsonpath='{.items[*].metadata.name}') ; echo $MAS_WORKSPACE_ID
##
ADMINDEV=$(oc get is ${MAS_WORKSPACE_ID}-admin -o jsonpath='{.status.publicDockerImageRepository}') ; echo $ADMINDEV
##
UIDEV=$(oc get is ${MAS_WORKSPACE_ID}-ui -o jsonpath='{.status.publicDockerImageRepository}') ; echo $UIDEV
##
podman pull $REGISTRY/mas-${MAS_INSTANCE_ID}-manage/${MAS_WORKSPACE_ID}-admin --tls-verify=false
##
podman pull $REGISTRY/mas-${MAS_INSTANCE_ID}-manage/${MAS_WORKSPACE_ID}-ui --tls-verify=false
##
podman tag $ADMINDEV manage-admin-dev
##
podman tag $UIDEV manage-ui-dev
##
### Replace mas8 with your local directory.
##
(/home/admin/mas8/manage-dev/manage-developer)
cd ..
(/home/admin/mas8/manage-dev)
podman build --build-arg BUNDLE=ui --tag managedev:latest -f ./manage-developer/Containerfile
```
```
podman run -dt --name maximo_dev -p 9080:9080/tcp localhost/managedev:latest
podman start -a -i maximo_dev
```

Access local Maximo/Manage via the browser:

localhost:9080/maximo `maxadmin/maxadmin`

---

`Containerfile`

```Dockerfile
ARG BUNDLE=ui
## Use Admin image as the builder
FROM localhost/manage-admin-dev AS admin

ARG BUNDLE
USER root

## Add customizations
# ADD applications /opt/IBM/SMP/maximo/applications
# WORKDIR /opt/IBM/SMP/maximo/applications/maximo
# RUN ls -l

## Replace server.xml
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/deployment/maximo-$BUNDLE/maximo-$BUNDLE-server/
RUN mv server.xml server-oidc.xml
RUN mv server-dev.xml server.xml

## Replace web.xml for maximo-all maximo-x
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/maximo-x/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-guest.xml  web-guest-oidc.xml
RUN mv web-dev.xml  web.xml
RUN mv web-guest-dev.xml  web-guest.xml

## Replace web.xml for maximo-all maximouiweb
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/maximouiweb/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml


## Replace web.xml for maximo-all maxrestweb
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/maxrestweb/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml


## Replace web.xml for maximo-all mboweb
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/mboweb/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml


## Replace web.xml for maximo-all meaweb
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/meaweb/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml

## Replace web.xml for maximo-cron
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-cron/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml

## Replace web.xml for maximo-mea maxrestweb
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-mea/maxrestweb/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml


## Replace web.xml for maximo-mea meaweb
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-mea/meaweb/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml

## Replace web.xml for maximo-report
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-report/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml

## Replace web.xml for maximo-ui
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-ui/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-dev.xml  web.xml

## Replace web.xml for maximo-x
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-x/webmodule/WEB-INF/
RUN mv web.xml web-oidc.xml
RUN mv web-guest.xml  web-guest-oidc.xml
RUN mv web-dev.xml  web.xml
RUN mv web-guest-dev.xml  web-guest.xml


## create the EAR file
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default
RUN ./maximo-$BUNDLE.sh
WORKDIR /opt/IBM/SMP/maximo/deployment/was-liberty-default/deployment/maximo-$BUNDLE/maximo-$BUNDLE-server/apps/
RUN ls -l

# Build the final image
FROM localhost/manage-${BUNDLE}-dev

ARG BUNDLE
RUN ls -l config/apps

## Copy the new EAR file from the admin builder
COPY --from=admin /opt/IBM/SMP/maximo/deployment/was-liberty-default/deployment/maximo-$BUNDLE/maximo-$BUNDLE-server/apps/*.war /config/apps
RUN ls -l config/apps

# Set the env vars - Database params need to be edited
# Manage project - Secret: internal-maximoproperty-secret
# Update your credentials and the IP address.
ENV DB_SSL_ENABLED=nossl
ENV MXE_DB_URL='jdbc:db2://192.168.252.121:31253/bludb'
ENV MXE_DB_SCHEMAOWNER=maximo
ENV MXE_DB_DRIVER='com.ibm.db2.jcc.DB2Driver'
ENV MXE_SECURITY_OLD_CRYPTO_KEY=VobmOvFQWGJippTiQlhphMGP
ENV MXE_SECURITY_CRYPTO_KEY=VobmOvFQWGJippTiQlhphMGP
ENV MXE_SECURITY_CRYPTOX_KEY=BormFhXcoHEmHXGEojUCesHz
ENV MXE_SECURITY_OLD_CRYPTOX=BormFhXcoHEmHXGEojUCesHz
ENV MXE_USEAPPSERVERSECURITY=0
ENV MXE_DB_USER=db2inst1
ENV MXE_DB_PASSWORD=<your database password>
ENV MXE_MASDEPLOYED=0
ENV LC_ALL=en_US.UTF-8

USER 1001
```
