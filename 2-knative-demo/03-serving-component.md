### The Serving component

The Serving component revolves around the concept of a **Service** (for clarity, it's called **KService**
throughout, because that term is vastly overloaded in the Kubernetes space). A KService is a higher level
construct that describes how an application is built and then how it's deployed in Knative.

A KService definition looks like this:

```sh
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: dumpy
  namespace: myproject
spec:
  runLatest:
    configuration:
      build:
        source:
          git:
            url: https://github.com/openshift-cloud-functions/openshift-knative-application
            revision: v1
        template:
          name: openshift-builds
          arguments:
          - name: IMAGE_STREAM
            value: golang:1.11
          - name: IMAGE
            value: "dumpy:latest"
          - name: NAME
            value: dumpy-build
      revisionTemplate:
        metadata:
          annotations:
            alpha.image.policy.openshift.io/resolve-names: "*"
        spec:
          containerConcurrency: 3
          container:
            imagePullPolicy: Always
            image: docker-registry.default.svc:5000/myproject/dumpy:latest
```

It's very apparent that the `spec.runLatest.configuration.build` part is a one-to-one copy of the Build
manifest we created above. If we apply this specification, Knative will go ahead and build the image
through the capabilities described above. Once that's done it'll go ahead and deploy a **Revision**,
an immutable snapshot of an application. Think of it as the **Configuration** of the application at a
specific point in time.

The `revisionTemplate` part of the KService specification describes how the Pods that will contain the
application will be created and deployed. In this case, it's going to pull the image that's built
from the OpenShift internal registry.

We create the KService by, you guessed it, applying the file through `oc`:

``oc apply -f serving/010-service.yaml``{{execute}}

Now the build will eventually start running. You can see through the OpenShift console, how a job is created
that orchestrates the build.  Go back to your OpenShift Web Console tab/window and on the left-hand side choose the
Application menu and choose Pods.  This will show you the pods being created for the build.

Eventually, an OpenShift Build is created and builds the image, as can be seenon the Builds page.  To see the build
on the OpenShift Web Console, go to the left-hand side and choose the Builds menu.

Once the build finishes, Knative will produce a couple of entities to actually deploy the application.
The KService consists of two parts: A **Route** and a **Configuration**. The `configuration` is directly apparent
through the respective part in the YAML file above, the Route is only implicitly there.

A Route makes the KService available under a hostname (see the `curl` example below). A Configuration is a description
of the application we're deploying. It contains a `revisionTemplate`, which hints to the fact that a Configuration
generates a new **Revision** for each change to the Configuration. A Revision is an immutable snapshot of a
Configuration and gives each deployed configuration a unique name. The Revision then generates a plain Kubernetes
**Deployment**, which in turn generates the **Pods** that generate the **Containers** for our application.

In short:

1. A Route makes our application available under a hostname.
2. A Configuration generates a Revision generates a Deployments generates Pods generates our Container.

Now, to see that the service is actually running, we're going to send a request against it. To do so,
we'll get the domain of the KService:

``oc get kservice``{{execute}}

And we will need the ingress gateway IP address which we can get via:

``export IP_ADDRESS=$(oc get svc knative-ingressgateway --namespace istio-system --output 'jsonpath={.status.loadBalancer.ingress[0].ip}')``{{execute}}

``echo $IP_ADDRESS``{{execute}}

Now we combine this together with the following command:

``curl -H "Host: dumpy.myproject.example.com" "http://${IP_ADDRESS}/health"``{{execute}}

If this works, you are ready to move on to the next step.
