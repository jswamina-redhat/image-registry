# OpenShift Container Platform Image Registry

## Purpose

This documentation will outline OCP Image Registry, how to push and maintain images, and how to scan images with Red Hat Advanced Cluster Security (ACS).

## Background

OpenShift Image Registry is an out-of-the-box, built-in container registry managed by the Image Registry Operator. It runs inside the openshift-image-registry namespace. The registry automatically handles container image management, storage, and build outputs for applications deployed on the Red Hat OpenShift Container Platform.

## Pushing Images

To push container images to the internal image registry in OpenShift, you need to expose the registry route, authenticate using your OpenShift credentials, tag your local image, and execute the push command using Podman or Docker.

Step 1: Expose the Internal Registry Route

>Expose the default route

```
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge`
```

>Get the registry host address and save it to a variable

```
export OCP_REGISTRY=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
echo $OCP_REGISTRY
```

Step 2: Authenticate to the Registry

```
podman login -u <user> -p $(oc whoami -t) --tls-verify=false $OCP_REGISTRY
```

Step 3: Target Your OpenShift Project

Make sure you have an active project or create a new namespace where your image will reside. You must also ensure an ImageStream is ready to track your image tags.

>Switch to your project (e.g., "my-project")

```
oc project my-project
```

>Create an ImageStream matching your image name

```
oc create imagestream my-app
```

Step 4: Tag and Push Your Local Image

Tag your local image using the format `${OCP_REGISTRY}/${PROJECT_NAME}/${IMAGE_NAME}:${TAG}`

>Tag the local image

```
podman tag local-image:latest $OCP_REGISTRY/my-project/my-app:v1
```

>Push to the OpenShift registry

```
podman push --tls-verify=false $OCP_REGISTRY/my-project/my-app:v1
```

Once completed, you can verify your image upload directly in the OpenShift Web Console under `Builds > ImageStreams` or by running `oc get imagestream my-app`

## Maintaing Images 

OpenShift image registry maintenance focuses primarily on reclaiming disk space and removing obsolete container images through automatic pruning, manual pruning, and hard pruning.

Because the internal registry tracks images using metadata (ImageStreams) and stores actual layers as blobs, standard object deletion does not automatically free backend disk space.

>Automatic Image Pruning

OpenShift manages automatic maintenance through the Image Pruner Operator. You can configure how often the pruner runs and how many historical images to retain by editing the cluster's custom resource (CR).

```
oc edit imagepruners.imageregistry.operator.openshift.io/cluster
```

The primary parameters to configure are as follows:

1. `schedule`: A cron expression determining when the job runs (e.g., `0 0 * * *` for midnight)
2. `keepTagRevisions`: The number of historical revisions to retain per image tag (default is `5`)
3. `keepYoungerThanDuration`: Do not delete images younger than this duration (e.g., `60m` or `240h`)

>Manual Image Pruning

If your storage is nearing capacity, you can manually trigger a prune operation using the OpenShift CLI (oc). This requires a user with the `system:image-pruner` cluster role

Step 1: Run a dry run to review what would be deleted without making structural changes

```
oc adm prune images --keep-tag-revisions=3 --keep-younger-than=240h
```

Step 2: Commit the deletions

```
oc adm prune images --keep-tag-revisions=3 --keep-younger-than=240h --confirm
```

>Hard Pruning (Blob Reclamation)

Sometimes, running `oc adm prune images` cleans up the OpenShift API tracking objects but leaves orphaned data blobs inside the underlying registry storage directory (`/registry`). To force the registry to reclaim this space, you must run a hard prune directly inside a registry pod

Step 1: Switch the registry to Read-Only mode to prevent push errors or database corruption

```
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"readOnly":true}}'

```

Step 2: Remote shell into a registry pod

```
oc rsh -n openshift-image-registry deployment/image-registry
```

Step 3: Execute the internal storage pruner

1. Check what will be removed: `registry garbage-collect /etc/docker/registry/config.yml --dry-run`

2. Delete the blobs permanently: `registry garbage-collect /etc/docker/registry/config.yml`

Step 4: Restore Read-Write mode

```
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"readOnly":false}}'
```

>Diagnostics & Troubleshooting

If maintenance jobs aren't clearing space, verify your usage and underlying storage metrics

1. Check actual storage utilization: 

```
oc -n openshift-image-registry rsh deployment/image-registry df -h /registry
```

2. Check Registry Operator Health

```
oc get clusteroperator image-registry
```

## Scanning Images with Red Hat Advancec Cluster Security (ACS)

To scan images in the internal OpenShift image registry using Red Hat Advanced Cluster Security (ACS), you need to configure an image integration within the ACS console so that its scanner (Scanner/StackRox) has permissions to pull and assess the internal images

Step 1: Create a Service Account for ACS

>Run the following command to create a service account inside the openshift-image-registry namespace:

```
oc create sa acs-registry-scanner -n openshift-image-registry
```

>Grant the service account the registry-viewer role so it can pull images

```
oc policy add-role-to-user registry-viewer system:serviceaccount:openshift-image-registry:acs-registry-scanner
```

>Create authentication token for `ServiceAccount`

```
oc create token acs-registry-scanner -n openshift-image-registry --duration=8760h
```

Step 2: Retrieve the Authentication Token

ACS will require this token to authenticate against the OpenShift registry.

1. Extract the token value from the secret linked to the service account

```
oc describe secret acs-registry-scanner -n openshift-image-registry
```

2. Copy the token string provided

Step 3: Configure the Integration in the ACS Portal

You will need to declare the OpenShift registry as a verified source inside the Red Hat Advanced Cluster Security Portal.

1. Log into your ACS Web Portal and navigate to `Platform Configuration` → `Integrations`

2. Scroll to the `Image Integrations` section and select `Generic Docker Registry`

3. Click New Integration and fill out the details
    1. Integration name: `Internal OpenShift Registry` (or any descriptive name)
    2. Endpoint: `image-registry.openshift-image-registry.svc:5000` (or your externally exposed registry route if scanning outside the cluster)
    3. Username: `acs-registry-scanner`
    4. Password: [Paste the Service Account Token you copied in Step 2]

4. Click `Test` to ensure connection validity, then click `Create / Save`

Step 4: Run an Image Scan Using the `roxctl` CLI

0. Create token in RHACS Portal
    1. Log in to your RHACS portal
    2. Go to `Platform Configuration` → `Integrations`
    3. Scroll down to the Authentication Tokens category and click on API Token
    4. Click `Generate Token`
    5. Enter a descriptive Name for your token and select an appropriate Role
    6. Click `Generate`
    7. Copy the generated token immediately and store it securely. You will not be shown this token again.

1. Set security variables

```
export ROX_CENTRAL_ENDPOINT="<central_host>:<port>"
export ROX_API_TOKEN="your-acs-api-token"
```

2. Execute an image check or full vulnerability scan

```
roxctl image check --image image-registry.openshift-image-registry.svc:5000/my-project/my-image:latest
```


### References

[Registry Overview](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/registry/registry-overview)

[Image Registry Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/registry/configuring-registry-operator)

[Pruning Objects](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/building_applications/pruning-objects)

[Using Red Hat Advanced Cluster Security with the OpenShift Registry](https://www.redhat.com/en/blog/using-red-hat-advanced-cluster-security-with-the-openshift-registry)

[Image Scanning](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_security_for_kubernetes/4.11/html/roxctl_cli/image-scanning-by-using-the-roxctl-cli-1)