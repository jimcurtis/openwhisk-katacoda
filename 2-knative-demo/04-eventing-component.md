### The Eventing component

The previous steps showed you how to invoke a KService from an HTTP request, in that case submitted via curl.
Knative Eventing is all about how you can invoke those applications in response to other events such as those received from message brokers or external applications.
In this part of the demo we are going to show how you can receive Kubernetes platform events and route those to a Knative Serving application.

Knative Eventing is built on top of three primitives:
* Event Sources
* Channels
* Subscriptions

**Event Sources** are the components that receive the external events and forward them onto **Sinks** which can be a **Channel**.
Out of the box we have Channels that are backed by Apache Kafka, GCPPubSub and a simple in-memory channel.
**Subscriptions** are used to connect Knative Serving application to a Channel so that it can respond to the events that the channel emits.

Let's take a look at some of those resources in more detail.

We have some yaml files prepared which describe the various resources, firstly the channel.

```sh
apiVersion: eventing.knative.dev/v1alpha1
kind: Channel
metadata:
  name: testchannel
spec:
  provisioner:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: ClusterChannelProvisioner
    name: in-memory-channel
```
Here we can see that we've got a channel named `testchannel` and it is an `in-memory-channel` which is what we're going to use for this demo - in production we would probably use Apache Kafka.
Let's deploy that so that we can use it in a later stage.

``oc apply -f eventing/010-channel.yaml``{{execute}}

Next let's take a look at the EventSource.

```sh
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: ContainerSource
metadata:
  name: urbanobservatory-event-source
spec:
  image: docker.io/markusthoemmes/knative-websocket-eventsource
  args: 
    - '--source=wss://api.usb.urbanobservatory.ac.uk/stream'
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: testchannel
```

This is starting to get a little bit more interesting. This EventSource is a so called `ContainerSource`. As such, it runs a container
based off the image given and instructs it to send its event to the sink described in the YAML. In this case, this container happens
to be a Websocket connector and we want all events coming from the specified Websocket server to be forwarded to the given sink. That
sink is the channel that we created before in this case.

This source in particular will emit IoT events from the buildings of the University of Newcastle, how cool is that? We'll allow it to
actually reach the defined host by setting up matching egress policies:

``oc apply -f eventing/020-egress.yaml``{{execute}}

If we now apply our source YAML, we will see a pod created which is an instance of the source we defined above.

``oc apply -f eventing/021-source.yaml``{{execute}}

Now, go over to your OpenShift Web Console tab/window to see created deployment for the source.

The EventSource is up and running and the final piece of the Knative Eventing is how we wire everything together.
This is done via a Subscription.

```sh
apiVersion: eventing.knative.dev/v1alpha1
kind: Subscription
metadata:
  name: testevents-subscription
  namespace: myproject
spec:
  channel:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: testchannel
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: dumpy
```

This Subscription again references the `testchannel` and also defines a `Subscriber`, in this case the Serving application `dumpy` we built earlier.
This basically means that we have subscribed the previously built application to the `testchannel`.
Once we apply the `Subscription` any Kubernetes platform events from the `myproject` namespace will be routed to the `dumpy` application.
Behind the scenes there is a `SubscriptionController` which is doing the wiring for us.

``oc apply -f eventing/030-subscription.yaml``{{execute}}

And by that, events coming through our source are now dispatched via a channel to the service that we created in the 
beginning of this tutorial. We can actually see the events by having a look at the logs of our application.  First, let's
see if the pod is running.

``oc get pods``{{execute}}

Then, let's use the `oc logs` command to show the logs for the container within the pod.

``oc logs -c user-container --since=1m $(oc get pods | grep -m1 -E "dumpy.*deployment.*Running" | awk '{print $1}')``{{execute}}

And you can't watch the deployments via the OpenShift Web Console if you go to that tab/window and click on the Montoring
tab.  You will see the Pods as well as the Deployments.
