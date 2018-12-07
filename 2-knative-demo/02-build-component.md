To prepare for this next step we need to clone the demo repo.  So, let's go into our myproject directory and clone the 
repo:

``cd myproject``{{execute}}

``git clone https://github.com/openshift-cloud-functions/demos.git``{{execute}}

Now, go down in the the directory where we will execute demo-related commands from:

``cd demos/knative-kubecon``{{execute}}

### Setting up access rights

To be able to access everything we need for the demo, we'll need to add certain rights to the `default` ServiceAccount
in our namespace. For the Build part it needs CRUD access to all OpenShift Build related entities (Build, BuildConfig,
ImageStream).

To set those up, run:

``oc apply -f build/000-rolebinding.yaml``{{execute}}

### The Build component

The Build component in Knative is not so much a utility to build images themselves. It rather provides primitives,
to be able to string together the tools you want to do your image build. In a sense, it's an abstraction layer above
all the tools out there to build an image. In our case, the most prominent example is OpenShift's own build capability,
so this will show you, how we can implement a Knative Build by the means of an OpenShift Build.

Knative provides a mechanism called **BuildTemplates**, where you define a blueprint for a build, which contains the
arguments that that build might need to do its job of building the image in the end. An example of such a template can
be seen below. This template allows building images through Knative Build based on OpenShift's Build capability, so you
can use the tools you're used to while taking full advantage of Knative Build on top of it.

```sh
apiVersion: build.knative.dev/v1alpha1
kind: BuildTemplate
metadata:
  name: openshift-builds
spec:
  parameters:
  - name: IMAGE
    description: The name of the image to push
  - name: NAME
    description: Build configuration name
  - name: IMAGE_STREAM
    description: The image stream to use as input for the build
  - name: TO_DOCKER
    description: Push the image to a Docker repository or not (true by default)
    default: "false"
  - name: DIRECTORY
    description: The directory containing the app
    default: /workspace
  - name: OC_BUILDER_IMAGE
    description: The name of the builder image to use
    default: docker.io/vdemeester/kobw-builder:0.1.1
  steps:
  - name: kobw-create-or-update
    image: "${OC_BUILDER_IMAGE}"
    args: ["create", "--name=${NAME}", "--image=${IMAGE}", "--image-stream=${IMAGE_STREAM}", "--to-docker=${TO_DOCKER}", "."]
    workingDir: "${DIRECTORY}"
    env:
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
  - name: kobw-run
    image: "${OC_BUILDER_IMAGE}"
    args: ["run", "--name=${NAME}"]
    env:
      - name: NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
```

To achieve the desired effect, this template wraps an OpenShift Build entity to perform the build of the desired application.
To "install" that template, we need to `oc apply` it by:

``oc apply -f build/010-build-template.yaml``{{execute}}

This on its own will do nothing. It only defines a template for a build to reference. A **Build**, as seen below, then 
includes such a references and provides it with the arguments needed to perform the build.

```sh
apiVersion: build.knative.dev/v1alpha1
kind: Build
metadata:
  name: oc-build-1
spec:
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
```

In this particular case, the source code is taken from a git repository and its release `v1` is checked out.
The arguments to the template then define that we want to build a Golang based application and want
to name the image `dumpy:latest`.

Now we could go ahead and run this build on its own by applying it like the template above and using
`oc apply -f build/020-build.yaml`, but it's much more interesting to see how it's stringed together
with creating a deployment in the same step. After all, an image on it's own is worth nothing if its
not deployed.
