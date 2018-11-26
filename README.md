# Cardiff AppDev Meetup - Microservices on Openshift using Fuse

## Overview

- ActiveMQ (enabled with Prometheus statistics)
- Camel Microservice to Generate orders to ActiveMQ and call an API
- Generate an API kickstarted with Apicurio
- Build an API to process generated orders into a database

## Prerequisites

- [Red Hat CDK](https://developers.redhat.com/products/cdk/download/)
- `git clone https://github.com/welshstew/cardiff-appdev-openshift.git`
- [jq](https://stedolan.github.io/jq/) - install however :)


## ActiveMQ Setup

In order to get ActiveMQ up and running:

Later on we well require the following queues, so we will explicity ensure these are created:

. incomingOrders
. incomingOrders.uk
. incomingOrders.us
. incomingOrders.other

The only reason we are building an image here and not using the AMQ one from the [Red Hat container catalog](https://access.redhat.com/containers/) is because we need
to make a slight modification to ensure that the prometheus agent is embedded in the container and we are able to scrape those statistics.  This uses source2image, as a basic explaination i'm overriding the bash scripts in the container and adding some extra files (prometheus jar).

The template itself is based on the application templates found at [jboss-openshift/application-templates](https://github.com/jboss-openshift/application-templates/)

```
cd amq63-prom-sti
oc new-project cardiff
oc import-image jboss-amq-63 --from=registry.access.redhat.com/jboss-amq-6/amq63-openshift --confirm -n openshift
oc new-build registry.access.redhat.com/jboss-amq-6/amq63-openshift:latest~https://github.com/welshstew/cardiff-appdev-openshift.git --context-dir=amq63-prom-sti --name=amq63-prom-sti
oc create -f amq63-basic-prom.json
oc new-app --template=amq63-basic-prom -p MQ_QUEUES=incomingOrders,incomingOrders.uk,incomingOrders.us,incomingOrders.other -p IMAGE_STREAM_NAMESPACE=cardiff -p MQ_USERNAME=amqadmin -p MQ_PASSWORD=amqadmin

```
We should now have an ActiveMQ up and running, we can see this through the jolokia enabled openshift console, and also we are able to open a remote shell into the pod and check that prometheus statistics are exposed.

```
oc get pods -l app=amq63-basic-prom -o json | jq -r '.items[0].metadata.name'
AMQ_PODNAME=$(oc get pods -l app=amq63-basic-prom -o json | jq -r '.items[0].metadata.name')
oc exec $AMQ_PODNAME -- curl localhost:9779

...
minishift console - show amq UI
```


## Creating our Order Generator Application

For this we will create the application with Camel using an archetype.  The archetpes can be viewed here: [Red Hat Fuse Archetypes](https://maven.repository.redhat.com/ga/io/fabric8/archetypes/archetypes-catalog/)

```
mvn org.apache.maven.plugins:maven-archetype-plugin:RELEASE:generate -DinteractiveMode=false -DgroupId=com.nullendpoint -DartifactId=fuse-order-generator -Dversion=1.0-SNAPSHOT -DarchetypeGroupId=org.jboss.fuse.fis.archetypes -DarchetypeArtifactId=spring-boot-camel-amq-archetype -DarchetypeVersion=2.2.195.redhat-000024 
```

The contents of the maven archetype generation can be found in fuse-order-generator/start.

The code was modified in fuse-order-generator/complete from the archetype generated project with the following:

- order files: are now json instead of xml (src/main/resources/data) and OrderGenerator java class
- message routing is changed to use json path (camel-context.xml)
- messages are routed to the  (camel-context.xml)
- application.properties changed to connect to activeMQ
- config.yml added in src/main/resources for prometheus statistics
- pom.xml changed to add in prometheus jar and jsonpath