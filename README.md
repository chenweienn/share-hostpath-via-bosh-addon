# share hostPath

## Pre-requisition

* [BOSH runtime config](https://bosh.io/docs/terminology/#addon)

## Procedure

1. upload [os-conf release](https://bosh.io/releases/github.com/cloudfoundry/os-conf-release?all=1) to BOSH director

```
bosh upload-release --sha1 7ef05f6f3ebc03f59ad8131851dbd1abd1ab3663 \
  https://bosh.io/d/github.com/cloudfoundry/os-conf-release?v=22.1.0
```
2. Prepare config-share-hostpath.yml

```
cat << EOF > config-share-hostpath.yml
releases:
- name: os-conf
  version: 22.1.0
 
addons:
- name: share-hostpath
  jobs:
  - name: pre-start-script
    release: os-conf
    properties:
      script: |-
        #!/bin/bash
        mount --make-rshared /
  include:
    instance_groups: [ worker ]
EOF
```

3. Update the runtime config into BOSH director

```
bosh update-runtime-config --name=share-hostpath ./config-share-hostpath.yml
bosh configs | grep share-hostpath
```

4. Apply the runtime config to all TKGI k8s clusters by running the **Upgrade all clusters errand** in Ops Manager UI. If you want to apply the runtime config to single cluster, steps are as follows.

```
# identify the BOSH deployment name of the TKGI k8s cluster
bosh deployments --column name --column "team(s)"

# retrieve the manifest of the cluster
bosh -d service-instance_GUID manifest > service-instance_GUID.yml

# re-deploy the cluster
bosh -d service-instance_GUID deploy service-instance_GUID.yml
```

5. Once the deployment completes successfully, you could verify that the runtime config **share-hostpath** has been associated with the clusters.

```
bosh curl "/deployment_configs?deployment[]=service-instance_GUID" | jq .
```

6. Verify if `/` is shared.

```
bosh -d service-instance_GUID ssh worker -c "sudo cat /proc/self/mountinfo | grep '/ / '"
```

7. To revert the change.

```
# delete the bosh runtime config
bosh delete-config --type runtime --name share-hostpath
bosh configs | grep share-hostpath

# make / private for each TKGI k8s cluster
bosh -d service-instance_GUID ssh worker -c "sudo mount --make-private /"
```
It is OK to defer the running of the **Upgrade all clusters errand** or `bosh deploy ...`, in order to remove the pre-start-script job added by the runtime config.
