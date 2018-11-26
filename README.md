# Cardiff AppDev Meetup - Microservices on Openshift using Fuse

## Overview

- ActiveMQ (enabled with Prometheus statistics)
- Camel Microservice to Generate orders to ActiveMQ and call an API
- Generate an API kickstarted with Apicurio
- Build an API to process generated orders into a database

## Prerequisites

. [Red Hat CDK](https://developers.redhat.com/products/cdk/download/)
. `git clone https://github.com/welshstew/cardiff-appdev-openshift.git`

## ActiveMQ Setup

In order to get ActiveMQ up and running:

Later on we well require the following queues, so we will explicity ensure these are created:

. incomingOrders
. incomingOrders.uk
. incomingOrders.us
. incomingOrders.other

```
cd amq63-prom-sti
oc new-project cardiff
oc import-image jboss-amq-63 --from=registry.access.redhat.com/jboss-amq-6/amq63-openshift --confirm -n openshift
oc new-build registry.access.redhat.com/jboss-amq-6/amq63-openshift:latest~https://github.com/welshstew/cardiff-appdev-openshift.git --context-dir=amq63-prom-sti
oc create -f amq63-basic-prom.json
oc new-app --template=amq63-basic-prom -p MQ_QUEUES=incomingOrders,incomingOrders.uk,incomingOrders.us,incomingOrders.other -p IMAGE_STREAM_NAMESPACE=cardiff

```