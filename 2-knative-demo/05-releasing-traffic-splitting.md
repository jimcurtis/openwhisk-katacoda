### Releasing and traffic splitting

If an application contains a bug, we'll want to fix that as soon as possible and roll out the new version as quickly and 
safe as possible. To do so, we'll rely on a concept called *canary release*. To achieve that, one wants to deploy a new 
version of an application and then gradually shift more and more traffic to the new version, to controllably verify it's 
working as intended. Shifting traffic gradually to a new version is called *traffic splitting*.

In this case, there is already a fixed version of the application available as `v2` (we used `v1`) above. To update our 
service we need to:

1. Instruct it not to run the latest version of our application (indicated by `runLatest` in the KService-YAML above)
2. Update the application's version to `v2`

That can be achieved through the following changes to the service definition:

```sh
diff
7c7,9
<   runLatest:
---
>   release:
>     revisions: ["dumpy-00001", "dumpy-00002"]
>     rolloutPercent: 0
13c15
<             revision: v1
---
>             revision: v2
```

This tells the Knative system to release from `dumpy-00001` (the current revision) to `dumpy-00002` (the canary
revision, the number is strictly increasing) and for now we want a rollout ratio of 0 because our new version still needs
to be built etc.

Apply these changes via:

``oc apply -f serving/011-service-update.yaml``{{execute}}

After the build is finished and the pods of the new revision have successfully started, we can start instructing Knative 
to actually send a portion of our traffic to the new version. To do so, we simply change `rolloutPercent` in our service 
definition to the desired value, let's make it `50` for now.

```sh
diff
9c9
<     rolloutPercent: 0
---
>     rolloutPercent: 50
```

Apply these changes via:

``oc apply -f serving/012-service-traffic.yaml``{{execute}}

The system will now evenly divide the traffic between the two deployed versions. Using Kiali, we can verify that the 
traffic to the new version is indeed not causing any errors.

Since we've now verified that the new version should indeed be rolled out completely, we can go ahead and move 100% 
of the traffic over. We do that by making "dumpy-00002" our current and only revision, and drop "dumpy-00001" completely. 
Since we're not rolling out anything, we set `rolloutPercent` to 0.

```sh
diff
8,9c8,9
<     revisions: ["dumpy-00001", "dumpy-00002"]
<     rolloutPercent: 50
---
>     revisions: ["dumpy-00002"]
>     rolloutPercent: 0
```

Apply these changes via:

``oc apply -f serving/013-service-final.yaml``{{execute}}

We've now successfully exchanged `v1` of our application with `v2` in a completely controlled manner and could've 
rolled back immediately at any point in time.
