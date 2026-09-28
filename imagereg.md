# OpenShift Container Platform Image Registry

## Purpose

This documentation will outline OCP Image Registry, how to push and maintain images, and how to scan images with Red Hat Advanced Cluster Security (ACS).

## Background

OpenShift Image Registry is an out-of-the-box, built-in container registry managed by the Image Registry Operator. It runs inside the `openshift-image-registry` namespace. The registry automatically handles container image management, storage, and build outputs for applications deployed on the Red Hat OpenShift Container Platform.

## Enabling the Image Registry

To enable the internal OpenShift Image Registry, you must change its management state from Removed to Managed and configure persistent storage. On platforms like bare metal the registry is disabled by default until storage is provisioned

### Provisioning & Configuring Storage

The registry requires shared storage that supports ReadWriteMany (RWX) access.

Step 1: Create PVC

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rwx-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: nfs-client # Replace with your cluster's RWX-compatible StorageClass
```

Step 2: Edit Registry Configuration

Edit the Image Registry Operator configuration resource (`oc edit configs.imageregistry.operator.openshift.io cluster`) to set `managementState: Managed` and assign your PVC claim. The operator will then automatically deploy the registry pods. 

Step 3: Verify the Registry Status

Check the ClusterOperator and pod status in the `openshift-image-registry` namespace to confirm the registry is healthy and running.

### Configuring Service Account & Token

Step 1: Create Service Account

>CLI

`oc create sa registry-sa -n <namespace>`

>YAML Manifest

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: registry-sa
  namespace: <namespace>
```

Step 2: Grant Registry Permissions

>Pull access (Read-only)

```
oc policy add-role-to-user system:image-puller system:serviceaccount:<namespace>:registry-sa -n <namespace>
```

>Push/Build access (Read/Write):

```
oc policy add-role-to-user system:image-builder system:serviceaccount:<namespace>:registry-sa -n <namespace>
```

Step 3: Generate a Long-Lived Token Secret


>sa-token.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: registry-sa-token
  namespace: <namespace>
  annotations:
    kubernetes.io/service-account.name: registry-sa
type: kubernetes.io/service-account-token
```

```
oc apply -f sa-token.yaml
```

Step 4: Authenticate to the OpenShift Image Registry

>Retrieve Token
```
TOKEN=$(oc get secret registry-sa-token -n <namespace> -o jsonpath='{.data.token}' | base64 -d)
```

>Expose Registry route
```
oc patch config.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"defaultRoute":true}}'
REGISTRY_HOST=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
```

>Authenticate using Podman
```
podman login -u serviceaccount -p $TOKEN $REGISTRY_HOST
```

Note: The username `-u` must be literally `serviceaccount`

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



### References

[Registry Overview](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/registry/registry-overview)

[Image Registry Operator](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/registry/configuring-registry-operator)

[Pruning Objects](https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/building_applications/pruning-objects)