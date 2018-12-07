This is a Knative on OpenShift demo as given at KubeCon. It walks you through on what Knative has to
offer:

1. It builds an application through Knative's Build component (which makes use of the existing OpenShift
Build mechanic underneath).
2. The built image is then deployed as a Knative Service, which means it scales automatically, even down
to nothing as we'll see.
3. We wire an EventSource emitting IoT events from the Newcastle University to our application through 
Knative's Eventing capabilities.
4. We'll roll out a new version via a canary release.
5. The application will be scaled according to the needs of the incoming traffic.
