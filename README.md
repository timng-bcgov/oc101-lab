# oc101-lab

## 03 - Deployment

### Create an ImageStreamTag

```
# retagging the image stream from latest to dev
oc -n [-tools] tag rocketchat-[username]:latest rocketchat-[username]:dev

# Verify that the `dev` tag has been created
oc -n [-tools] get imagestreamtag/rocketchat-[username]:dev
```
### Importing Images to the deploy namespace

1. Re-tag the tools image tag into the new ImageStream

```
oc -n [-dev] tag [-tools]/rocketchat-[username]:dev rocketchat-[username]:dev
```

2. Modify the RocketChat deployment to point to the new ImageStream

```
oc -n [-dev] set image deployment/rocketchat-[username] rocketchat-[username]=rocketchat-[username]:dev
```

### Resolve the CrashLoopBackOff error by deploying the database

List available parameters of the template:

```
oc -n d8f105-dev process -f https://raw.githubusercontent.com/BCDevOps/devops-platform-workshops/master/101-lab/mongo-ephemeral-template.yaml --parameters=true
```

Create MongoDB deployment, service, and secret using the template:

```
 oc -n [dev] process -f https://raw.githubusercontent.com/BCDevOps/devops-platform-workshops/master/101-lab/mongo-ephemeral-template.yaml -p MONGODB_USER=dbuser MONGODB_PASSWORD=dbpass MONGODB_ADMIN_PASSWORD=admindbpass MONGODB_DATABASE=rocketchat MONGODB_NAME=mongodb-[username] -l ocp101=participant | oc -n [-dev] create -f - --dry-run=client
```

Remove the dry run:

```
 oc -n [dev] process -f https://raw.githubusercontent.com/BCDevOps/devops-platform-workshops/master/101-lab/mongo-ephemeral-template.yaml -p MONGODB_USER=dbuser MONGODB_PASSWORD=dbpass MONGODB_ADMIN_PASSWORD=admindbpass MONGODB_DATABASE=rocketchat MONGODB_NAME=mongodb-[username] -l ocp101=participant | oc -n [dev] create -f - 
```

Verify the MongoDB deployment is up by navigating to `Topology` in the OpenShift console details

### Add environment variables

```
oc -n [-dev] set env deployment/rocketchat-[username] "MONGO_URL=mongodb://dbuser:dbpass@mongodb-[username]:27017/rocketchat"
```

Navigate to `Topology` in the OpenShift web console and investigate your RocketChat deployment

### Create a Route for the app

```
oc -n [-dev] create route edge rocketchat-[username] --service=rocketchat-[username] --insecure-policy=Redirect
```

### Health Checks

Add a healthcheck for `readiness` and `liveness`:

```
oc -n [-dev] set probe deployment/rocketchat-[username] --readiness --get-url=http://:3000/ --initial-delay-seconds=15
```
A readiness check will be added to the deployment so that you no longer have a false positive of when the pod should be considered available. By default pods are considered to be 'ready' when the container starts up and the entrypoint script is running. This however is not useful for things like webservers or databases! Not only do you need the entrypoint script to run but you need to wait for the server to listen on a port.

## 04 - Exploring deployment options

### Using `oc explain`

Discover the fields that belong to a Deployment

```
oc explain deployment
```
## 05 - Resource requests and limits

View the resource limit ranges in place:

```
oc get LimitRange
oc -n [-dev] describe LimitRange default-limits
```
Which returns:

```
Name:       default-limits
Namespace:  d8f105-dev
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Container   cpu       -    -    50m              -              -
Container   memory    -    -    256Mi            -              -
```

## 06 - Application availability

### Scaling pods

To increase the availability of an application and defend against unplanned outages or planned maintenance tasks, an application must have multiple pods/instance running. For this reason, stateless applications are desirable as they do not require custom clustering configurations.

Stateful applications do not support "scaling pods" as a form of high availability. Such a stateful example would be the mongodb database. For this reason, this lab focuses on the rocketchat application which will function with multiple pods. Please refer to specific application documentaion for details on scalability support.


## 08 - Persistent storage

Up to this point, we've leveraged a single MongoDB pod with ephemeral storage. In order to maintain the application data, persistent storage is required.

https://kubernetes.io/docs/concepts/storage/volumes/

### RWO Storage

RWO storage is analogous to attaching a physical disk to a pod. For this reason, RWO storage cannot be mounted to more than 1 pod at the same time.

### RWX Storage

```
# Remove all volumes
oc -n d8f105-dev get deployment/mongodb-timng -o jsonpath='{.spec.template.spec.volumes[].name}{"\n"}' | xargs -I {} oc -n d8f105-dev set volumes deployment/mongodb-timng --remove '--name={}'
```

```
# Attach a new volume
oc -n d8f105-dev set volume deployment/mongodb-timng --add --name=mongodb-timng-data -m /var/lib/mongodb/data -t pvc --claim-name=mongodb-timng-file
```

```
# Use the Recreate deployment strategy
oc -n d8f105-dev patch deployment/mongodb-timng -p '{"spec":{"strategy":{"activeDeadlineSeconds":21600,"recreateParams":{"timeoutSeconds":600},"resources":{},"type":"Recreate"}}}'
```

## 09 - Persistent configurations

In cases where configurations need to change frequently or common configurations should be shared across deployments or pods, it is not ideal to build said configurations into the container image or maintain multiple copies of the configuration. OpenShift supports configMaps which can be a standalone object that is easily mounted into pods. In cases where the configuration file or data is sensitive in nature, OpenShift supports secrets to handle this sensitive data.

### Create and add ConfigMap to deployments



# Command reference

## Builds
```
oc project d8f105-tools
oc -n d8f105-tools new-build https://github.com/BCDevOps/devops-platform-workshops-labs/ --context-dir=apps/rocketchat --name=rocketchat-mattspencer
oc -n d8f105-tools logs -f bc/rocketchat-mattspencer
```
## Deployment
```
oc -n d8f105-tools tag rocketchat-mattspencer:latest rocketchat-mattspencer:dev
oc -n d8f105-dev new-app d8f105-tools/rocketchat-mattspencer:dev --name=rocketchat-mattspencer -l ocp101=participant
oc -n d8f105-dev set resources deployment/rocketchat-mattspencer --requests='cpu=500m,memory=512Mi' --limits='cpu=1000m,memory=1024Mi'
oc -n d8f105-tools policy add-role-to-user system:image-puller system:serviceaccount: d8f105-dev:default
oc scale deployment/rocketchat-mattspencer --replicas=0
oc scale deployment/rocketchat-mattspencer --replicas=1
oc -n d8f105-dev tag d8f105-tools/rocketchat-mattspencer:dev rocketchat-mattspencer:dev
oc -n d8f105-dev set image deployment/rocketchat-mattspencer rocketchat-mattspencer=rocketchat-mattspencer:dev
oc -n d8f105-dev new-app --search mongodb-ephemeral -l ocp101=participant
oc -n d8f105-dev new-app --template=openshift/mongodb-ephemeral -p MONGODB_VERSION=3.6 -p DATABASE_SERVICE_NAME=mongodb-mattspencer -p MONGODB_USER=dbuser -p MONGODB_PASSWORD=dbpass -p MONGODB_DATABASE=rocketchat --name=rocketchat-mattspencer -l ocp101=participant
oc -n d8f105-dev set env deployment/rocketchat-mattspencer "MONGO_URL=mongodb://dbuser:dbpass@mongodb-mattspencer:27017/rocketchat"
```
_STRETCH_ 

```
oc -n d8f105-dev rollout pause deployment/rocketchat-mattspencer 
oc -n d8f105-dev patch deployment/rocketchat-mattspencer -p '{"spec":{"template":{"spec":{"containers":[{"name":"rocketchat-mattspencer", "env":[{"name":"MONGO_USER", "valueFrom":{"secretKeyRef":{"key":"database-user", "name":"mongodb-mattspencer"}}}]}]}}}}'
oc -n d8f105-dev patch deployment/rocketchat-mattspencer -p '{"spec":{"template":{"spec":{"containers":[{"name":"rocketchat-mattspencer", "env":[{"name":"MONGO_PASS", "valueFrom":{"secretKeyRef":{"key":"database-password", "name":"mongodb-mattspencer"}}}]}]}}}}'
oc -n d8f105-dev set env deployment/rocketchat-mattspencer 'MONGO_URL=mongodb://$(MONGO_USER):$(MONGO_PASS)@mongodb-mattspencer:27017/rocketchat'
oc -n d8f105-dev rollout resume deployment/rocketchat-mattspencer 
oc -n d8f105-dev get deployment/rocketchat-mattspencer -o json | jq '.spec.template.spec.containers[].env'
```
## Create a Route for your Rocket.Chat App
```
oc -n d8f105-dev expose svc/rocketchat-mattspencer
OR
oc -n d8f105-dev create route edge rocketchat-mattspencer --service=rocketchat-mattspencer --insecure-policy=Redirect

oc -n d8f105-dev set probe deployment/rocketchat-mattspencer --readiness --get-url=http://:3000/ --initial-delay-seconds=15
oc -n d8f105-dev set env deployment/rocketchat-mattspencer TEST=BAR
```
## Resource Requests and Limits 
```
oc -n d8f105-dev set resources deployment/rocketchat-mattspencer --requests=cpu=65m,memory=100Mi --limits=cpu=65m,memory=100Mi
oc -n d8f105-dev rollout restart deployment/rocketchat-mattspencer
time oc -n d8f105-dev rollout restart deployment/rocketchat-mattspencer
oc -n d8f105-dev set resources deployment/rocketchat-mattspencer --requests=cpu=1000m,memory=512Mi --limits=cpu=1000m,memory=1024Mi
oc -n d8f105-dev rollout restart deployment/rocketchat-mattspencer
time oc -n d8f105-dev rollout restart deployment/rocketchat-mattspencer
oc -n d8f105-dev set resources deployment/rocketchat-mattspencer --requests=cpu=150m,memory=256Mi --limits=cpu=200m,memory=400Mi
```
## Application Availability 
```
oc -n d8f105-dev scale deployment/rocketchat-mattspencer --replicas=2
oc -n d8f105-dev get pods --field-selector=status.phase=Running -l deployment=rocketchat-mattspencer -o wide
oc -n d8f105-dev get pods -o wide | grep rocketchat-mattspencer
oc -n d8f105-dev get pods --field-selector=status.phase=Running -l deployment=rocketchat-mattspencer -o name | head -1 | xargs -I {} oc -n d8f105-dev delete {}
brew install watch
watch -dg -n 1 curl -fsSL https://rocketchat-mattspencer-d8f105-dev.apps.silver.devops.gov.bc.ca/api/info
oc -n d8f105-dev get pods --field-selector=status.phase=Running -l deployment=rocketchat-mattspencer -o name | head -1 | xargs -I {} oc -n d8f105-dev delete {}
watch -dg -n 1 curl -fsSL https://rocketchat-mattspencer-d8f105-dev.apps.silver.devops.gov.bc.ca/api/info
watch -dg -n 1 curl -fsSL https://rocketchat-mattspencer-dev.d8f105-dev.apps.silver.devops.gov.bc.ca/api/info
```

## Autoscaling
```
oc -n d8f105-dev autoscale deployment/rocketchat-mattspencer --min 1 --max 10 --cpu-percent=10
oc -n d8f105-dev delete hpa/rocketchat-mattspencer
```
## Persistent Storage 
```
oc -n d8f105-dev scale deployment/rocketchat-mattspencer deployment/mongodb-mattspencer --replicas=0
oc -n d8f105-dev scale deployment/mongodb-mattspencer --replicas=1
oc -n d8f105-dev scale deployment/rocketchat-mattspencer --replicas=1
oc -n d8f105-dev scale deployment/mongodb-mattspencer --replicas=0 
oc -n d8f105-dev rollout pause deployment/mongodb-mattspencer 
oc -n d8f105-dev get deployment/mongodb-mattspencer -o jsonpath='{.spec.template.spec.volumes[].name}{"\n"}' | xargs -I {} oc -n d8f105-dev set volumes deployment/mongodb-mattspencer --remove '--name={}'
oc -n d8f105-dev set volume deployment/mongodb-mattspencer --add --name=mongodb-mattspencer-data -m /var/lib/mongodb/data -t pvc --claim-name=mongodb-mattspencer-file
oc -n d8f105-dev scale deployment/mongodb-mattspencer --replicas=1 
oc -n d8f105-dev rollout resume deployment/mongodb-mattspencer
oc -n d8f105-dev scale deployment/rocketchat-mattspencer --replicas=0
oc -n d8f105-dev scale deployment/mongodb-mattspencer --replicas=0 
oc -n d8f105-dev rollout pause deployment/mongodb-mattspencer
oc -n d8f105-dev get deployment/mongodb-mattspencer -o jsonpath='{.spec.template.spec.volumes[].name}{"\n"}' | xargs -I {} oc -n d8f105-dev set volumes deployment/mongodb-mattspencer --remove '--name={}'
oc -n d8f105-dev set volume deployment/mongodb-mattspencer --add --name=mongodb-mattspencer-data -m /var/lib/mongodb/data -t pvc --claim-name=mongodb-mattspencer-file
oc -n d8f105-dev patch deployment/mongodb-mattspencer -p '{"spec":{"strategy":{"activeDeadlineSeconds":21600,"recreateParams":{"timeoutSeconds":600},"resources":{},"type":"Recreate"}}}'
oc -n d8f105-dev rollout resume deployment/mongodb-mattspencer
oc -n d8f105-dev rollout latest deployment/mongodb-mattspencer
oc -n d8f105-dev scale deployment/mongodb-mattspencer --replicas=1 
oc -n d8f105-dev scale deployment/rocketchat-mattspencer --replicas=1
oc -n d8f105-dev delete pvc/mongodb-mattspencer-file-rwx
```
## Persistent Configurations
```
- name: rocketchat-mattspencer-volume
          configMap:
            name: rocketchat-mattspencer-configmap

oc -n d8f105-dev describe secret rocketchat-mattspencer-secret
oc -n d8f105-dev get secret rocketchat-mattspencer-secret -o yaml
```
## Event Streams

## Debugging Containers
```
oc -n [-dev] logs -f <pod name>
oc -n d8f105-dev scale deployment/mongodb-mattspencer --replicas=0 
oc -n d8f105-dev rollout restart deployment/rocketchat-mattspencer
oc -n d8f105-dev get pods
oc -n d8f105-dev debug <podname>
oc -n d8f105-dev get pods  | grep rocketchat-mattspencer
oc -n d8f105-dev port-forward rocketchat-mattspencer-597bd65475-nlrmw 8000:3000
```







