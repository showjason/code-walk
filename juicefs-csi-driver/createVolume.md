## Juicefs csi driver 创建 PV

这部分代码走读是针对 pod mount

当创建了 PVC 之后，Kubernetes 的 PVcontroller 发现新的 PVC 被创建，并确认这个 PVC 属于 out-of-tree，便会给这个 PVC 加上 `annotation volume.kubernetes.io/storage-provisioner: csi.juicefs.com` 和 `volume.beta.kubernetes.io/storage-provisioner: csi.juicefs.com`，后者将被废弃。

external-provisioner 通过如下代码设置 annotation

```
func (ctrl *PersistentVolumeController) setClaimProvisioner(ctx context.Context, claim *v1.PersistentVolumeClaim, provisionerName string) (*v1.PersistentVolumeClaim, error) {
	if val, ok := claim.Annotations[storagehelpers.AnnStorageProvisioner]; ok && val == provisionerName {
		// annotation is already set, nothing to do
		return claim, nil
	}

	// The volume from method args can be pointing to watcher cache. We must not
	// modify these, therefore create a copy.
	claimClone := claim.DeepCopy()
	// TODO: remove the beta storage provisioner anno after the deprecation period
	logger := klog.FromContext(ctx)
	metav1.SetMetaDataAnnotation(&claimClone.ObjectMeta, storagehelpers.AnnBetaStorageProvisioner, provisionerName)
	metav1.SetMetaDataAnnotation(&claimClone.ObjectMeta, storagehelpers.AnnStorageProvisioner, provisionerName)
	updateMigrationAnnotations(logger, ctrl.csiMigratedPluginManager, ctrl.translator, claimClone.Annotations, true)
	newClaim, err := ctrl.kubeClient.CoreV1().PersistentVolumeClaims(claim.Namespace).Update(ctx, claimClone, metav1.UpdateOptions{})
	if err != nil {
		return newClaim, err
	}
	_, err = ctrl.storeClaimUpdate(logger, newClaim)
	if err != nil {
		return newClaim, err
	}
	return newClaim, nil
}
```

这时  Juicefs csi driver 的 sidecar `external-provisioner` 的 infomer 机制会发现这个 PVC，并对比 PVC 的 annotation 的值与 csi driver 是否相同，这里的 `driverName` 是通过调用 GetPluginInfo 获取的 。如果相同，则创建一个 CreateVolumeRequest, 通过 RPC 请求调用 juicfs csi controller 的 CreateVolume，创建对应的 PV (这里省略了部分代码)。

```
func (p *csiProvisioner) Provision(ctx context.Context, options controller.ProvisionOptions) (*v1.PersistentVolume, controller.ProvisioningState, error) {
	claim := options.PVC
	provisioner, ok := claim.Annotations[annStorageProvisioner]
	if !ok {
		provisioner = claim.Annotations[annBetaStorageProvisioner]
	}
	if provisioner != p.driverName && claim.Annotations[annMigratedTo] != p.driverName {
		// The storage provisioner annotation may not equal driver name but the
		// PVC could have annotation "migrated-to" which is the new way to
		// signal a PVC is migrated (k8s v1.17+)
		return nil, controller.ProvisioningFinished, &controller.IgnoredError{
			Reason: fmt.Sprintf("PVC annotated with external-provisioner name %s does not match provisioner driver name %s. This could mean the PVC is not migrated",
				provisioner,
				p.driverName),
		}
	}

	....
	req := result.req
	volSizeBytes := req.CapacityRange.RequiredBytes
	pvName := req.Name
	provisionerCredentials := req.Secrets

	createCtx := markAsMigrated(ctx, result.migratedVolume)
	createCtx, cancel := context.WithTimeout(createCtx, p.timeout)
	defer cancel()
	rep, err := p.csiClient.CreateVolume(createCtx, req)
	if err != nil {
		// Giving up after an error and telling the pod scheduler to retry with a different node
		// only makes sense if:
		// - The CSI driver supports topology: without that, the next CreateVolume call after
		//   rescheduling will be exactly the same.
		// - We are working on a volume with late binding: only in that case will
		//   provisioning be retried if we give up for now.
		// - The error is one where rescheduling is
		//   a) allowed (i.e. we don't have to keep calling CreateVolume because the operation might be running) and
		//   b) it makes sense (typically local resource exhausted).
		//   isFinalError is going to check this.
		//
		// We do this regardless whether the driver has asked for strict topology because
		// even drivers which did not ask for it explicitly might still only look at the first
		// topology entry and thus succeed after rescheduling.

		....

	}
	....
	// According to CSI spec CreateVolume should be able to return capacity = 0, which means it is unknown. for example NFS/FTP

	.....

	pvReadOnly := false
	volCaps := req.GetVolumeCapabilities()
	// if the request only has one accessmode and if its ROX, set readonly to true
	// TODO: check for the driver capability of MULTI_NODE_READER_ONLY capability from the CSI driver
	....

	// Set annDeletionSecretRefName and namespace in PV object.

        ....

	// Set VolumeMode to PV if it is passed via PVC spec when Block feature is enabled
	....
	// Set FSType if PV is not Block Volume
	....
	klog.V(2).Infof("successfully created PV %v for PVC %v and csi volume name %v", pv.Name, options.PVC.Name, pv.Spec.CSI.VolumeHandle)

	....

	klog.V(5).Infof("successfully created PV %+v", pv.Spec.PersistentVolumeSource)
	return pv, controller.ProvisioningFinished, nil
}
```

CreateVolume 方法接收到请求之后，开始创建 PV。Juicefs 的 CreateVolume 方法没有实现具体的创建过程，也没有实现 csi 的 attach/detach 和 staging 功能，而是由 NodePublishVolume 创建了一个 mount pod 来实现 volume 的格式化和挂载。

创建 PV 的流程如下：

```
node.go
    func NodePublishVolume -> mount volume
	juicefs.go
	    func CreateTarget -> create target dir
            func JfsMount -> generate mount fs object(jfs)
                juicefs.go
                    func MountFs -> 判断是 pod 挂载，还是进程挂载
                      如果是 pod 挂载(mount_pod): pod_mount.go
				    func JMount -> 生成 mount pod，挂载点 和 pv 的 reference
                                        pod_mount.go
        				    func setMountLabel
					    func createOrAddRef
       					    func waitUtilMountReady
  	    func jfs.CreateVol -> 创建 Volume
  	    jfs.BindTarget -> 绑定 path
		 BindTarget: binding /jfs/pvc-2c02a96f-6683-41b9-9d00-242954bfd6ac-tzdxni/pvc-2c02a96f-6683-41b9-9d00-242954bfd6ac at /var/lib/kubelet/pods/4f2bd0db-159c-4370-86e8-58524750946c/volumes/kubernetes.io~csi/pvc-2c02a96f-6683-41b9-9d00-242954bfd6ac/mount
	    juicefs.SetQuota -> 代码返回 error，貌似不妥，是否可以优化
                 SetQuota cmd: /usr/local/bin/juicefs quota set ${metaurl} --path pvc-2c02a96f-6683-41b9-9d00-242954bfd6ac --capacity 10
```

mount pod 代码流程：

```
func (p *PodMount) JMount(ctx context.Context, appInfo *jfsConfig.AppInfo, jfsSetting *jfsConfig.JfsSetting) error {
	podName, err := p.genMountPodName(ctx, jfsSetting)
	if err != nil {
		return err
	}

	// set mount pod name in app pod
	if appInfo != nil && appInfo.Name != "" && appInfo.Namespace != "" {
		err = p.setMountLabel(ctx, jfsSetting.UniqueId, podName, appInfo.Name, appInfo.Namespace)
		if err != nil {
			return err
		}
	}
	// 创建 mount pod, 如果在同一个节点上，使用的是相同的 PV，则增加 reference
	err = p.createOrAddRef(ctx, podName, jfsSetting, appInfo)
	if err != nil {
		return err
	}
	// 等待挂载操作完成
	err = p.waitUtilMountReady(ctx, jfsSetting, podName)
	if err != nil {
		return err
	}
	if jfsSetting.CleanCache && jfsSetting.UUID == "" {
		// need set uuid as label in mount pod for clean cache
		uuid, err := p.GetJfsVolUUID(ctx, jfsSetting.Source)
		if err != nil {
			return err
		}
		err = p.setUUIDAnnotation(ctx, podName, uuid)
		if err != nil {
			return err
		}
	}
	return nil
}
```
