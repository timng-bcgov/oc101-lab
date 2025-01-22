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






