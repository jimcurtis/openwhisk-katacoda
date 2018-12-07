### Scaling Up & Down

For our demo, we have been using an event source from the University of Newcastle.  This source generates a vast and steady
stream of events for us to consume.  When we created the event subscription earlier, Knative scaled our application 
to fit the incoming volume dynamically.  Go to your OpenShift Web Console tab/window and you will see in the Monitoring
screen that the deployment of our app is showing 3 or 4 pods to handle the events.

The promise of Serverless is to only spin up enough pods to serve the traffic to our application. That means, if there is
no traffic our application should have zero pods. We can simulate that by removing the subscription of the channel to our
application:

``oc delete -f eventing/030-subscription.yaml``{{execute}}

Now go back to the OpenShift Web Console tab/window and you'll see the pods slowly disappearing until they even disappear
completely. 

Let's reinstantiate the subscription with this:

``oc apply -f eventing/030-subscription.yaml``{{execute}}

And go back to the OpenShift Web Console tab/window and watch the deployment scale back up.
