# oc101-lab

## Deployment

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
