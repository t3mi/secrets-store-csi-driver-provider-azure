---
type: docs
title: "Sync Mounted Content with Kubernetes Secret"
linkTitle: "Sync Mounted Content with Kubernetes Secret"
weight: 1
description: >
  How to sync mounted content with Kubernetes secret
---

<details>
<summary>Examples</summary>

- `SecretProviderClass`
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: azure-sync
spec:
  provider: azure
  secretObjects:                                 # [OPTIONAL] SecretObject defines the desired state of synced K8s secret objects
  - secretName: foosecret
    type: Opaque
    labels:                                   
      environment: "test"
    data: 
    - objectName: secretalias                    # name of the mounted content to sync. this could be the object name or object alias 
      key: username
  parameters:
    usePodIdentity: "true"                      
    keyvaultName: "$KEYVAULT_NAME"               # the name of the KeyVault
    objects: |
      array:
        - |
          objectName: $SECRET_NAME
          objectType: secret                     # object types: secret, key or cert
          objectAlias: secretalias
          objectVersion: $SECRET_VERSION         # [OPTIONAL] object versions, default to latest if empty
        - |
          objectName: $KEY_NAME
          objectType: key
          objectVersion: $KEY_VERSION
    tenantId: "tid"                             # the tenant ID of the KeyVault
``` 

- `Pod` yaml
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: nginx-secrets-store-inline
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
      - name: secrets-store01-inline
        mountPath: "/mnt/secrets-store"
        readOnly: true
  volumes:
    - name: secrets-store01-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "azure-sync"
```
</details>

### How to sync mounted content with Kubernetes secret

In some cases, you may want to create a Kubernetes Secret to mirror the mounted content. Use the optional `secretObjects` field to define the desired state of the synced Kubernetes secret objects.

> NOTE: Make sure the `objectName` in `secretObjects` matches the file name of the mounted content. This could be the object name or the object alias.

> The secrets will only sync once you *start a pod mounting the secrets*. Solely relying on the syncing with Kubernetes secrets feature thus does not work.


A `SecretProviderClass` custom resource should have the following components:
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: my-provider
spec:
  provider: azure                             
  secretObjects:                              # [OPTIONAL] SecretObject defines the desired state of synced K8s secret objects
  - data:
    - key: username                           # data field to populate
      objectName: foo1                        # name of the mounted content to sync. this could be the object name or the object alias
    secretName: foosecret                     # name of the Kubernetes Secret object
    type: Opaque                              # type of the Kubernetes Secret object e.g. Opaque, kubernetes.io/tls
...
```
> NOTE: Here is the list of supported Kubernetes Secret types: `Opaque`, `kubernetes.io/basic-auth`, `bootstrap.kubernetes.io/token`, `kubernetes.io/dockerconfigjson`, `kubernetes.io/dockercfg`, `kubernetes.io/ssh-auth`, `kubernetes.io/service-account-token`, `kubernetes.io/tls`.  

- Here is a sample [`SecretProviderClass` custom resource](https://github.com/kubernetes-sigs/secrets-store-csi-driver/blob/master/test/bats/tests/azure/azure_synck8s_v1alpha1_secretproviderclass.yaml) that syncs a secret from Azure Key Vault to a Kubernetes secret.
- To view an example of type `kubernetes.io/tls`, refer to the [ingress-controller-tls sample](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/ingress-tls.md)