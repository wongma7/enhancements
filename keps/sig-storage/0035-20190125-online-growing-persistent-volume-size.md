---
title: Online Growing Persistent Volume Size
authors:
  - "@mlmhl"
  - "@wongma7"
owning-sig: sig-xxx
participating-sigs:
  - sig-storage
reviewers:
  - "@gnufied"
  - "@jsafrane"
approvers:
  - "@childsb"
editor: TBD
creation-date: 2019-01-25
last-updated: 2019-01-25
status: implementable
see-also:
  - https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/grow-volume-size.md
  - https://github.com/kubernetes/community/pull/1535
replaces:
superseded-by:
---

# Online Growing Persistent Volume Size

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Summary](#summary)
* [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
* [Proposal](#proposal)
    * [User Stories [optional]](#user-stories)
      * [Story 1](#story-1)
      * [Story 2](#story-2)
    * [Notes](#notes)
    * [Implementation Details](#implementation-details)
    * [Risks and Mitigations](#risks-and-mitigations)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)

[Tools for generating]: https://github.com/ekalinin/github-markdown-toc

## Summary

This feature enables users to expand a volume's file system by editing a PVC without having to restart a pod using the PVC.

## Motivation

Release 1.10 only supports offline file system resizing for PVCs, as this operation is only executed inside the `MountVolume` operation in kubelet. If a resizing request was submitted after the volume was mounted, it won't be performed. This proposal's intent is to support online file system resizing for PVCs in kubelet.

### Goals

Enable users to increase the size of a PVC which is already in use (mounted). The user will update PVC to request a new size. Underneath we expect that kubelet will resize the file system for the PVC accordingly.

### Non-Goals

* Offline file system resizing is not included. If we find a volume needs file system resizing but is not mounted to the node yet, we will do nothing. This situation will be dealt with by the existing [offline file system resizing handler](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/grow-volume-size.md).

* Extending resize tools: we only support the most common file systems' offline resizing in current release, and we prefer to stay the same for online resizing: ext3, ext4, & xfs.

## Proposal

### User Stories

#### Story 1

* As a user I am running MySQL on a 100GB volume - but I am running out of space. I should be able to increase size of volume MySQL is using without losing all my data. (online and with data)

#### Story 2

* As a user I am running an application with a PVC. I should be able to resize the volume without losing data or mount point. (online and with data and without taking pod offline)

### Notes

- Currently we only support offline resizing for `xfs`, `ext3`, `ext4`. Online resizing of `ext3`,`ext4` was introduced in [Linux kernel-3.3](https://www.ibm.com/developerworks/library/l-33linuxkernel/), and `xfs` has always supported growing mounted partitions (in fact, currently there is no way to expand an unmounted `xfs` file system), so they are all safe for online resizing. If a user tries to expand a volume with other formats an error event will be reported for the pod using it.

- This feature is protected by an alpha feature gate `ExpandOnlinePersistentVolumes` in v1.11. We separate this feature gate from the offline resizing gate `ExpandPersistentVolumes`, if a user wants to enable this feature, `ExpandPersistentVolumes` must be enabled first.

### Implementation Details

The key point of online file system resizing is how kubelet discovers which  PVCs need file system resizing. We achieve this goal by reusing the reprocess mechanism of `VolumeManager`'s `DesiredStateOfWorldPopulator`. In detail, kubelet synchronizes pods' status periodically, and during each loop, `DesiredStateOfWorldPopulator` will reprocess each pod's volumes and add them to `ActualStateOfWorld`. If a volume refers to a PVC, it will fetch both PV and PVC objects from apiserver. So we can check online resize request on these latest fetched objects.

#### Online File System Resize in Kubelet

##### Volume Resize Request Detect

As mentioned before,`DesiredStateOfWorldPopulator` loops through all alive pods in `PodManager` and reprocess each pod's volumes periodically. If a volume refers to a `PersistentVolumeClaim`, it will fetch the latest PV and PVC objects from apiserver, see [here](https://github.com/kubernetes/kubernetes/blob/6b64c07bafd13f7a2f4396526f60a189d407e363/pkg/kubelet/volumemanager/populator/desired_state_of_world_populator.go#L439) and [here](https://github.com/kubernetes/kubernetes/blob/2f13ffb056488dd34c92a0e0e1c57bef0a45f27d/pkg/kubelet/volumemanager/populator/desired_state_of_world_populator.go#L485). The reason for this is we need the PV object to create `VolumeSpec`, and we need the PVC object to know which PV it is bound to.

With these latest objects, we can check whether a PVC requires an online resizing operation or not. A PVC requires online file system resizing if:

* `PVC.Status.Capacity` is less than  `PV.Spec.Capacity`
* `PVC.Status.Condition` has a `PersistentVolumeClaimFileSystemResizePending` condition

To mark a volume as file system resizing required, we add a `fsResizeRequired` field to `attachedVolume.mountedPod`, and add a `MarkFSResizeRequired` method to `ActualStateOfWorld`. If `DesiredStateOfWorldPopulator` discovers a volume requires file system resizing, it invokes `MarkFSResizeRequired` to set according `mountedPod.fsResizeRequired` to true.

`MarkFSResizeRequired` looks like this:

```go
func (asw *actualStateOfWorld) MarkFSResizeRequired(
	volumeName v1.UniqueVolumeName,
	podName volumetypes.UniquePodName) {
	if !utilfeature.DefaultFeatureGate.Enabled(features.ExpandOnlinePersistentVolumes) {
		// Do not perform online resizing if feature gate is disabled.
		return
	}
	asw.Lock()
	defer asw.Unlock()
	volumeObj, exist := asw.attachedVolumes[volumeName]
	if !exist {
		glog.Warningf("MarkFSResizeRequired for volume %s failed as volume not exist", volumeName)
		return
	}

	podObj, exist := volumeObj.mountedPods[podName]
	if !exist {
		glog.Warningf("MarkFSResizeRequired for volume %s failed "+
			"as pod(%s) not exist", volumeName, podName)
		return
	}

	volumePlugin, err :=
		asw.volumePluginMgr.FindExpandablePluginBySpec(volumeObj.spec)
	if err != nil || volumePlugin == nil {
		// Log and continue processing
		glog.Errorf(
			"MarkFSResizeRequired failed to FindPluginBySpec for pod %q (podUid %q) volume: %q (volSpecName: %q)",
			podObj.podName,
			podObj.podUID,
			volumeObj.volumeName,
			volumeObj.spec.Name())
		return
	}

	if volumePlugin.RequiresFSResize() {
		if !podObj.fsResizeRequired {
			glog.V(3).Infof("PVC volume %s(OuterVolumeSpecName %s) of pod %s requires file system resize",
				volumeName, podObj.outerVolumeSpecName, podName)
			podObj.fsResizeRequired = true
		}
		asw.attachedVolumes[volumeName].mountedPods[podName] = podObj
	}
}
```

And `DesiredStateOfWorldPopulator` uses a `checkVolumeFSResize` method to perform the operation described as above, which invokes inside `processPodVolumes` method:

```go
// checkVolumeFSResize checks whether a PVC mounted by the pod requires file
// system resize or not. If so, marks this volume as fsResizeRequired in ASW.
// - mountedVolumesForPod stores all mounted volumes in ASW, because online
//   volume resize only considers mounted volumes.
// - processedVolumesForFSResize stores all volumes we have checked in current loop,
//   because file system resize operation is a global operation for volume, so
//   we only need to check it once if more than one pod use it.
func (dswp *desiredStateOfWorldPopulator) checkVolumeFSResize(
	pod *v1.Pod,
	podVolume v1.Volume,
	pvc *v1.PersistentVolumeClaim,
	volumeSpec *volume.Spec,
	uniquePodName volumetypes.UniquePodName,
	mountedVolumesForPod map[volumetypes.UniquePodName]map[string]cache.MountedVolume,
	processedVolumesForFSResize sets.String) {
	if podVolume.PersistentVolumeClaim == nil {
		// Only PVC supports resize operation.
		return
	}
	uniqueVolumeName, exist := getUniqueVolumeNameFromASW(uniquePodName, podVolume.Name, mountedVolumesForPod)
	if !exist {
		// Volume not exist in ASW, we assume it hasn't been mounted yet. If it needs resize,
		// it will be handled as offline resize(if it indeed hasn't been mounted yet),
		// or online resize in subsequent loop(after we confirm it has been mounted).
		return
	}
	fsVolume, err := util.CheckVolumeModeFilesystem(volumeSpec)
	if err != nil {
		glog.Errorf("Check volume mode failed for volume %s(OuterVolumeSpecName %s): %v",
			uniqueVolumeName, podVolume.Name, err)
		return
	}
	if !fsVolume {
		glog.V(5).Infof("Block mode volume needn't to check file system resize request")
		return
	}
	if processedVolumesForFSResize.Has(string(uniqueVolumeName)) {
		// File system resize operation is a global operation for volume,
		// so we only need to check it once if more than one pod use it.
		return
	}
	if mountedReadOnlyByPod(podVolume, pod) {
		// This volume is used as read only by this pod, we don't perform resize for read only volumes.
		glog.V(5).Infof("Skip file system resize check for volume %s in pod %s/%s "+
			"as the volume is mounted as readonly", podVolume.Name, pod.Namespace, pod.Name)
		return
	}
	if volumeRequiresFSResize(pvc, volumeSpec.PersistentVolume) {
		dswp.actualStateOfWorld.MarkFSResizeRequired(uniqueVolumeName, uniquePodName)
	}
	processedVolumesForFSResize.Insert(string(uniqueVolumeName))
}
```

It is important to note that:

- We only perform online resizing for mounted volumes, it means that these volumes must exist in `ActualStateOfWorld`, so we fetch all mount volumes from ASW in advance (stored in input parameter `mountedVolumesForPod`), and skip volumes which don't exist in `mountedVolumesForPod`.

- We only perform online resizing for PVCs used as`Filesystem` mode, because `Block` mode volume will not be formatted with a file system.

- If a PVC is mounted as read only by a pod, we won't invoke `MarkFSResizeRequired` for this pod, to promise the semantic of `ReadOnly`.

- Although we add `fsResizeRequired` to `mountedPod`, actually file system resizing is a global operation for a volume, if more than one pod mounts the same PVC, we needn't invoke `MarkFSResizeRequired` for each pod. Instead, we only mark the first pod we processed. The input parameter `processedVolumesForFSResize` is introduced for this purpose.

- Since we support only `xfs` and `ext3/4`, we needn't worry about exotic file systems that can be attached/mounted to multiple nodes at the same time and require a resizing operation on a node, such as [gfs2](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/global_file_system_2/s1-manage-growfs).

##### File System Resizing Operation

Currently `Reconciler` runs a periodic loop to reconcile the desired state of the world with the actual state of the world by triggering attach, detach, mount, and unmount operations. Based on this mechanism, we can make `Reconciler` trigger file system resize operations too.

In each loop, for each volume in DSW, `Reconciler` checks whether it exists in ASW or not. If the volume exists and is marked as file system resizing required, ASW returns a `fsResizeRequiredError` error. Then `Reconciler` triggers a file system resizing operation for this volume. The code snippet looks like this:

```go
if cache.IsFSResizeRequiredError(err) {
	glog.V(4).Infof(volumeToMount.GenerateMsgDetailed("Starting operationExecutor.VolumeFileSystemResize", ""))
	err := rc.operationExecutor.VolumeFileSystemResize(
		volumeToMount.VolumeToMount,
		rc.actualStateOfWorld)
	if err != nil &&
		!nestedpendingoperations.IsAlreadyExists(err) &&
		!exponentialbackoff.IsExponentialBackoff(err) {
		// Ignore nestedpendingoperations.IsAlreadyExists and exponentialbackoff.IsExponentialBackoff errors, they are expected.
		// Log all other errors.
		glog.Errorf(volumeToMount.GenerateErrorDetailed("operationExecutor.VolumeFileSystemResize failed", err).Error())
	}
	if err == nil {
		glog.V(4).Infof(volumeToMount.GenerateMsgDetailed("operationExecutor.VolumeFileSystemResize started", ""))
	}
}
```

As can be seen from the above code, we add a `VolumeFileSystemResize` method to `OperationExecutor` and a `GenerateVolumeFSResizeFunc` method to `OperationGenerator`, which performs volume file system resizing operation.

We also add a `MarkVolumeAsResized` to `ActualStateOfWorldMounterUpdater`, which will be invoked after the file system resizing operation succeeds. `MarkVolumeAsResized` clears the `fsResizeRequired` flag of the according `mountedPod` to promise that no more resize operation will be performed.

The main work of `GenerateVolumeFSResizeFunc` is simply invoking the `resizeFileSystem` method and `MarkVolumeAsResized`, it looks like this:

```go
	resizeSimpleError, resizeDetailedError := og.resizeFileSystem(volumeToMount, volumeToMount.DevicePath, deviceMountPath, volumePlugin.GetPluginName())
	if resizeSimpleError != nil || resizeDetailedError != nil {
		return resizeSimpleError, resizeDetailedError
	}
	markFSResizedErr := actualStateOfWorld.MarkVolumeAsResized(volumeToMount.PodName, volumeToMount.VolumeName)
	if markFSResizedErr != nil {
		// On failure, return error. Caller will log and retry.
		return volumeToMount.GenerateError("VolumeFSResize.MarkVolumeAsResized failed", markFSResizedErr)
	}
	return nil, nil
```

- If the file system resizing operation succeeds, `resizeFileSystem` will change `PVC.Status.Capacity` to the desired volume size in `PV.Spec.Capacity`, and remove the `PersistentVolumeClaimFileSystemResizePending` condition from `PVC.Status.Conditions`, then this resizing request is marked as finished.

- If any operation fails inside `GenerateVolumeFSResizeFunc`, an event will be added to the according pod and an error will be returned to `Reconciler`, `Reconciler` will log this error and retry the resizing operation later in the next loop.


### Risks and Mitigations

- inadvertant online resize when only offline desired - mitigation TBD

## Graduation Criteria

### Alpha to Beta

- e2e tests

[umbrella issues]: https://github.com/kubernetes/kubernetes/issues/42752

## Implementation History

- 1.11: alpha
