


external-provisioner/cmd/csi-provisioner/csi-provisioner.go

```
func main() {
    ....
    csiProvisioner := ctrl.NewCSIProvisioner(
    ....
    )
    ....
    provisionController = controller.NewProvisionController(
        clientset,
	provisionerName,
	csiProvisioner,
	provisionerOptions...,
    )  
    ....
    provisionController.Run(ctx)
    ....
}

```


```

func (ctrl *ProvisionController) Run(_ <-chan struct{}) {
	...
	run := func(ctx context.Context) {
	    ....
	    for i := 0; i < ctrl.threadiness; i++ {
		go wait.Until(ctrl.runClaimWorker, time.Second, context.TODO().Done())
		go wait.Until(ctrl.runVolumeWorker, time.Second, context.TODO().Done())
	    }
	    ....
	}
}

func (ctrl *ProvisionController) runVolumeWorker() {
	for ctrl.processNextVolumeWorkItem() {
	}
}

// processNextVolumeWorkItem processes items from volumeQueue
func (ctrl *ProvisionController) processNextVolumeWorkItem() bool {
	obj, shutdown := ctrl.volumeQueue.Get()

	if shutdown {
		return false
	}

	err := func(obj interface{}) error {
		defer ctrl.volumeQueue.Done(obj)
		var key string
		var ok bool
		if key, ok = obj.(string); !ok {
			ctrl.volumeQueue.Forget(obj)
			return fmt.Errorf("expected string in workqueue but got %#v", obj)
		}

		if err := ctrl.syncVolumeHandler(key); err != nil {
			if ctrl.failedDeleteThreshold == 0 {
				glog.Warningf("Retrying syncing volume %q, failure %v", key, ctrl.volumeQueue.NumRequeues(obj))
				ctrl.volumeQueue.AddRateLimited(obj)
			} else if ctrl.volumeQueue.NumRequeues(obj) < ctrl.failedDeleteThreshold {
				glog.Warningf("Retrying syncing volume %q because failures %v < threshold %v", key, ctrl.volumeQueue.NumRequeues(obj), ctrl.failedDeleteThreshold)
				ctrl.volumeQueue.AddRateLimited(obj)
			} else {
				glog.Errorf("Giving up syncing volume %q because failures %v >= threshold %v", key, ctrl.volumeQueue.NumRequeues(obj), ctrl.failedDeleteThreshold)
				// Done but do not Forget: it will not be in the queue but NumRequeues
				// will be saved until the obj is deleted from kubernetes
			}
			return fmt.Errorf("error syncing volume %q: %s", key, err.Error())
		}

		ctrl.volumeQueue.Forget(obj)
		return nil
	}(obj)

	if err != nil {
		utilruntime.HandleError(err)
		return true
	}

	return true
}

// syncVolumeHandler gets the volume from informer's cache then calls syncVolume
func (ctrl *ProvisionController) syncVolumeHandler(key string) error {
	volumeObj, exists, err := ctrl.volumes.GetByKey(key)
	....
	return ctrl.syncVolume(volumeObj)
}
// 检查是否有需要删除的 Volume
func (ctrl *ProvisionController) syncVolume(obj interface{}) error {}

```


```
func (ctrl *ProvisionController) deleteVolumeOperation(volume *v1.PersistentVolume) error {
	operation := fmt.Sprintf("delete %q", volume.Name)
	glog.Info(logOperation(operation, "started"))
	....
	// 调用 csi 的 RPC DeleteVolume 
	err = ctrl.provisioner.Delete(volume)
	if err != nil {
		if ierr, ok := err.(*IgnoredError); ok {
			// Delete ignored, do nothing and hope another provisioner will delete it.
			glog.Info(logOperation(operation, "volume deletion ignored: %v", ierr))
			return nil
		}
		// Delete failed, emit an event.
		glog.Error(logOperation(operation, "volume deletion failed: %v", err))
		ctrl.eventRecorder.Event(volume, v1.EventTypeWarning, "VolumeFailedDelete", err.Error())
		return err
	}

	glog.Info(logOperation(operation, "volume deleted"))

	// 删除 persistent volume
	if err = ctrl.client.CoreV1().PersistentVolumes().Delete(volume.Name, nil); err != nil {
		// Oops, could not delete the volume and therefore the controller will
		// try to delete the volume again on next update.
		glog.Info(logOperation(operation, "failed to delete persistentvolume: %v", err))
		return err
	}

	if ctrl.addFinalizer {
		if len(newVolume.ObjectMeta.Finalizers) > 0 {
			// Remove external-provisioner finalizer

			// need to get the pv again because the delete has updated the object with a deletion timestamp
			newVolume, err := ctrl.client.CoreV1().PersistentVolumes().Get(volume.Name, metav1.GetOptions{})
			if err != nil {
				// If the volume is not found return, otherwise error
				if !apierrs.IsNotFound(err) {
					glog.Info(logOperation(operation, "failed to get persistentvolume to update finalizer: %v", err))
					return err
				}
				return nil
			}
			finalizers := make([]string, 0)
			for _, finalizer := range newVolume.ObjectMeta.Finalizers {
				if finalizer != finalizerPV {
					finalizers = append(finalizers, finalizer)
				}
			}

			// Only update the finalizers if we actually removed something
			if len(finalizers) != len(newVolume.ObjectMeta.Finalizers) {
				newVolume.ObjectMeta.Finalizers = finalizers
				if _, err = ctrl.client.CoreV1().PersistentVolumes().Update(newVolume); err != nil {
					if !apierrs.IsNotFound(err) {
						// Couldn't remove finalizer and the object still exists, the controller may
						// try to remove the finalizer again on the next update
						glog.Info(logOperation(operation, "failed to remove finalizer for persistentvolume: %v", err))
						return err
					}
				}
			}
		}
	}

	glog.Info(logOperation(operation, "persistentvolume deleted"))

	glog.Info(logOperation(operation, "succeeded"))
	return nil
}

```

构造 delete volume request, 调用 CSI RPC DeleteVolume。

```
func (p *csiProvisioner) Delete(volume *v1.PersistentVolume) error {
	if volume == nil {
		return fmt.Errorf("invalid CSI PV")
	}

	if volume.Spec.CSI == nil {
		return fmt.Errorf("invalid CSI PV")
	}

	volumeId := p.volumeHandleToId(volume.Spec.CSI.VolumeHandle)

	rc := &requiredCapabilities{}
	if err := p.checkDriverCapabilities(rc); err != nil {
		return err
	}
	// 构造 DeleteVolumeRequest
	req := csi.DeleteVolumeRequest{
		VolumeId: volumeId,
	}

	// get secrets if StorageClass specifies it
	storageClassName := util.GetPersistentVolumeClass(volume)
	if len(storageClassName) != 0 {
		if storageClass, err := p.scLister.Get(storageClassName); err == nil {
			// Resolve provision secret credentials.
			provisionerSecretRef, err := getSecretReference(provisionerSecretParams, storageClass.Parameters, volume.Name, &v1.PersistentVolumeClaim{
				ObjectMeta: metav1.ObjectMeta{
					Name:      volume.Spec.ClaimRef.Name,
					Namespace: volume.Spec.ClaimRef.Namespace,
				},
			})
			....

			credentials, err := getCredentials(p.client, provisionerSecretRef)
			if err != nil {
				// Continue with deletion, as the secret may have already been deleted.
				klog.Errorf("Failed to get credentials for volume %s: %s", volume.Name, err.Error())
			}
			req.Secrets = credentials
		} else {
			klog.Warningf("failed to get storageclass: %s, proceeding to delete without secrets. %v", storageClassName, err)
		}
	}
	....
	// 调用 CSI RPC DeleteVolume
	_, err = p.csiClient.DeleteVolume(ctx, &req)

	return err
}
```
