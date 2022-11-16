---
layout: blog
title: "Retroactive default StorageClass"
date: 2022-11-11
slug: retroactive-default-storage-class
---

**Author:** Roman Bednář (Red Hat)

In Kubernetes 1.25 a new Alpha feature was introduced to improve the way default StorageClass is assigned to a PersistentVolumeClaim (PVC). It is no longer required to create a default StorageClass first and PVC second in order to assign the class and any PVCs without StorageClass can be updated later. The feature will be Beta in Kubernetes 1.26.

See [Official Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#retroactive-default-storageclass-assignment) for more details.

## Why did StorageClass assignment need improvements

When talking about PVCs and default StorageClasses you might think "It already works, when I create PVC it gets the default class just fine!" and that is correct because there already is a feature that causes admission controller to assign a class to **new** PVCs which is documented [here](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass).

But what if there was no default StorageClass at the time of PVC creation? Users end up with a PVC that will never get a class and so no storage will be provisioned, the PVC is somewhat "stuck" at this point. From high level point of view there were two main use cases which might result in "stuck" PVCs and cause problems later down the road. Let's take a closer look at each of them.

### Changing default StorageClass did not have a good solution

   There are two options admins had when they wanted to change default StorageClass:
   1. Creating the new StorageClass as default before removing the old one which results in having two defaults for a short period of time. If a user creates a PersistentVolumeClaim at this point with storageClassName set to `nil` (which means default StorageClass) Kubernetes 1.26+ will choose the newest default StorageClass and assign it to this PVC.


   2. Remove the old default first and create new default StorageClass which results in having no default for a short period of time. If user creates a PersistentVolumeClaim with storageClassName set to `nil` (which means default StorageClass) the PVC will be in `Pending` state forever, then user has to fix this by deleting the PVC a creating it again once default StorageClass is available.

Looking at the scenarios above it is clear that neither of them is optimal.

### Resource ordering during cluster installation

If cluster installation tool needs to create resources that require storage (e.g. image registry) it was difficult to get the ordering right because any Pods that require storage and rely on default StorageClass presence will fail to create if there is no default at that point yet.

## What changed

We've changed PersistentVolume (PV) controller in KCM, so it can assign default StorageClass to unbound PersistentVolumeClaims that have storageClassName set to `nil`. In order to allow the value change PersistentVolumeClaim admission in the API server also changed to allow changing the value from `nil` to actual StorageClass name.

### `nil` vs `""` for storageClassName - does it matter?

Before this feature was introduced those values were equal in terms of behavior and any PersistentVolumeClaim with `nil` or `""` storageClassName value would bind to existing PersistentVolume resource with storageClassName also set to `nil` or `""`.

With this new feature enabled we wanted to maintain this behavior but also be able to update the StorageClass name. With these constraints in mind the feature changes the semantics of `nil` if default StorageClass is present to always mean "Give me a default" and `""` to mean "Give me PersistentVolume that also has `""` StorageClass name".

In other words we need to distinguish two main cases, either default StorageClass is not present in which case there's no behavior change or where there is a default in which case PVCs with `nil` won't bind to PVs anymore and instead will be updated to have the StorageClass.

The tables below show all these cases to better describe when PVC binds and when it's StorageClass gets updated.

| Without default class        | PVC storageClassName == `""` | PVC storageClassName == `nil` |
|------------------------------|------------------------------|-------------------------------|
| PV storageClassName == `""`  | binds                        | binds                         |
| PV without storageClassName  | binds                        | binds                         |


| With default class          | PVC storageClassName == `""` | PVC storageClassName == `nil` |
|-----------------------------|------------------------------|-------------------------------|
| PV storageClassName == `""` | binds                        | class updates                 |
| PV without storageClassName | binds                        | class updates                 |

> NOTE: `storageClassName` with `nil` value in PVC spec is equal to omitting the `storageClassName` in the spec.


## How to use it

You don't have to perform any configuration steps - the feature will work "out of the box" once it's GA.

If you want to test the feature in Alpha you need to enable FeatureGates in `kube-controller-manager` and `kube-apiserver`:

```bash
--feature-gates="...,RetroactiveDefaultStorageClass=true"
```

### Test drive

If you would like to see the feature in action and verify it works fine in your cluster here's what you can try:

1. We'll use a very simple PersistentVolumeClaim.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: csi-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

2. Create the PersistentVolumeClaim when there is no default StorageClass. The PVC won't provision or bind (unless there is an existing, suitable PV already present) and will remain in `Pending` state.
```bash
$ kc get pvc
NAME      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
csi-pvc   Pending   
```

3. Configure one StorageClass as default.

```bash
$ kc patch sc/csi-hostpath-sc -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
storageclass.storage.k8s.io/csi-hostpath-sc patched
```

4. Verify that PersistentVolumeClaims is now provisioned correctly and was updated retroactively with new default StorageClass.

```bash
$ kc get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
csi-pvc   Bound    pvc-06a964ca-f997-4780-8627-b5c3bf5a87d8   1Gi        RWO            csi-hostpath-sc   87m
```

### New metrics

To help indicate that the feature is working as expected we also introduced new `retroactive_storageclass_total` metric to show how many times PV controller attempted to update PersistentVolumeClaim and `retroactive_storageclass_errors_total` to show how many of those attempts failed.

## Getting involved

If you would like to share feedback with us, you can do so on our [public Slack channel](https://app.slack.com/client/T09NY5SBT/C09QZFCE5).

We always welcome new contributors so if you would like to get involved you can join our [Kubernetes Storage Special-Interest-Group](https://github.com/kubernetes/community/tree/master/sig-storage) (SIG).

Special thanks to all the contributors that provided great reviews, shared valuable insight and helped implement this feature (alphabetical order):

- Deep Debroy ([ddebroy](https://github.com/ddebroy))
- Jan Šafránek ([jsafrane](https://github.com/jsafrane/))
- Joe Betz ([jpbetz](https://github.com/jpbetz))
- Jordan Liggitt ([liggitt](https://github.com/liggitt))
- Michelle Au ([msau42](https://github.com/msau42))
- Seokho Son ([seokho-son](https://github.com/seokho-son))
- Shannon Kularathna ([shannonxtreme](https://github.com/shannonxtreme))
- Tim Bannister ([sftim](https://github.com/sftim))
- Tim Hockin ([thockin](https://github.com/thockin))
- Wojciech Tyczynski ([wojtek-t](https://github.com/wojtek-t))
- Xing Yang ([xing-yang](https://github.com/xing-yang))
