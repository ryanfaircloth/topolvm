# Limitations

<!-- Created by VSCode Markdown All in One command: Create Table of Contents -->
- [StorageClass Reclaim Policy](#storageclass-reclaim-policy)
- [Pod without PVC](#pod-without-pvc)
- [Capacity Aware Scheduling May Go Wrong](#capacity-aware-scheduling-may-go-wrong)
- [Snapshots Can Be Created Only for Thin Volumes](#snapshots-can-be-created-only-for-thin-volumes)
- [Snapshots Can Be Restored Only on the Same Node with the Source Volume](#snapshots-can-be-restored-only-on-the-same-node-with-the-source-volume)
- [Use lvcreate-options at Your Own Risk](#use-lvcreate-options-at-your-own-risk)

## StorageClass Reclaim Policy

TopoLVM does not care about `Retain` [reclaim policy](https://kubernetes.io/docs/concepts/storage/storage-classes/#reclaim-policy)
because CSI volumes can be referenced only via PersistentVolumeClaims.

Ref: https://kubernetes.io/docs/concepts/storage/volumes/#csi

> The `csi` volume type does not support direct reference from Pod and may
> only be referenced in a Pod via a `PersistentVolumeClaim` object.

If you delete a PVC whose corresponding PV has `Retain` reclaim policy, the corresponding `LogicalVolume` resource and the LVM logical volume are *NOT* deleted. If you delete this `LogicalVolume` resource after deleting the PVC, the related LVM logical volume is also deleted.

## Pod without PVC

TopoLVM expects that PVCs are created in advance of their Pods.
However, the TopoLVM webhook does not block the creation of a Pod when there are missing PVCs for the Pod.
This is because such usages are valid in other StorageClasses and the webhook cannot identify the StorageClasses without PVCs.
For such Pods, TopoLVM's extended scheduler will not work.

The typical usage of TopoLVM is using StatefulSet with volumeClaimTemplate.

## Capacity Aware Scheduling May Go Wrong

Node storage capacity annotation is not updated in TopoLVM's extended scheduler.
Therefore, when multiple pods requesting TopoLVM volumes are created at once, the extended scheduler cannot reference the exact capacity of the underlying LVM volume group.

Note that pod scheduling is also affected by the amount of CPU and memory.
Because of this, this problem may not be observable.

## Snapshots Can Be Created Only for Thin Volumes

It is because we now implemented the feature only for thin volumes.
For thin volumes, it is easy to be implemented, however for thick volumes, it may be hard.
For example, a thick volume which has its snapshots cannot be expanded without inactivating the snapshots.

## Snapshots Can Be Restored Only on the Same Node with the Source Volume

Since TopoLVM uses LVM's snapshot feature, TopoLVM's snapshots can be restored only on the same node with the source logical volume.

## Use lvcreate-options at Your Own Risk

TopoLVM does not check the `lvcreate-options` that can optionally be added to a device-class.
Therefore it cannot take them into consideration when scheduling, or do sanity checks for them.
It is up to the user to make sure that these arguments make sense and work with the VG in question.
For example, with `--type=raid1`, the VG must have at least 2 PVs to be able to create any LVs.

Note also that the options may affect the "actual" available capacity.
With `--type=raid1`, each LV will take up twice the normal space.
You may want to tweak the `spare-gb` setting to avoid some issues with this.

**Example**

There is one VG `raid-vg` with two PVs (`disk1` and `disk2`).
Then we create the following LVMd config:

```yaml
device-classes:
  - name: "raid1"
    volume-group: "raid-vg"
    spare-gb: 100
    lvcreate-options:
      - "--type=raid1"
```

If we now ask for a volume with this device class it will be created using `lvcreate` with `--type=raid1`.
This means that the data will be mirrored on the two disks.
Notice the following:

1. The VG *must* have at least two PVs in it with enough capacity or volume creation will fail.
   This is a requirement coming from the RAID configuration and is up to the user to take into account when creating the VG and device-class.
2. Since the data is mirrored on the two disks, it takes up twice as much space.
   If we ask for a volume with 1 GB capacity, it will use 1 GB on each disk, i.e. 2 GB total of the VG.
   TopoLVM does not know about this so it will not be considered when doing scheduling decisions.

To help with the scheduling we can set `spare-gb` to the size of one disk.
For example, if `disk1` and `disk2` are 100 GB each, we can set `spare-gb` to 100 GB so that the reported capacity from TopoLVM would be 100 GB.
This is the "real" capacity that is available when all LVs are created with `--type=raid1`.
However, this will not be correct once we create some volumes, since they use up some capacity.
For example, we ask for a 50 GB volume.
This volume will use 100 GB in total since it is of type RAID1.
The VG has 200 GB total and 100 GB spare configured so TopoLVM will now consider this VG full.

For more details please see [this proposal](./proposals/lvcreate-options.md).
