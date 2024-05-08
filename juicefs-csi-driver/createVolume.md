## Juicefs csi driver 创建 PV

这部分代码走读是基于 pod mount

当创建了 PVC 之后，Kubernetes 的 PVcontroller 会检查到并检查这个 PVC 存储类型是 out-of-tree，会给这个 PVC 加上 annotation。这时  external-provisioner 会发现这个 PVC，然后创建一个 CreateVolumeRequest, 通过 RPC 请求调用 juicfs csi controller 的 CreateVolume，创建对应的 PV。

CreateVolume 方法接收到请求之后，开始创建 PV。Juicefs 的 CreateVolume 方法没有实现具体的创建过程。。。

没有 attach/detach 和 staging 阶段，直接跳到 publishing 阶段。

创建 PV 的流程如下：

```
node.go
    func NodePublishVolume -> mount volume
	juicefs.go
	    func CreateTarget -> create target dir
            func JfsMount -> generate mount fs object(jfs)
                juicefs.go
                    func MountFs -> 判断是 pod 挂载，还是进程挂载
                      如果是 pod 挂在(mount_pod): pod_mount.go
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
