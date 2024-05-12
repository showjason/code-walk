# Create Volume

入口函数 external-provisioner/cmd/csi-provisioner/csi-provisioner.go

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

主要的创建过程 vendor sig v9 controller.go

```
    func (ctrl *ProvisionController) Run(ctx context.Context)
	...
	go wait.Until(func() { ctrl.runClaimWorker(ctx) }, time.Second, ctx.Done())
	...

    func (ctrl *ProvisionController) runClaimWorker(ctx context.Context)
	....
	for ctrl.processNextClaimWorkItem(ctx)
	....

    func (ctrl *ProvisionController) processNextClaimWorkItem(ctx context.Context)
	....
	if err := ctrl.syncClaimHandler(ctx, key);
	....

    func (ctrl *ProvisionController) syncClaimHandler(ctx context.Context, key string)
	....
	return ctrl.syncClaim(ctx, claimObj)

    func (ctrl *ProvisionController) syncClaim(ctx context.Context, obj interface{})
	....
	status, err := ctrl.provisionClaimOperation(ctx, claim)
	....

    func (ctrl *ProvisionController) provisionClaimOperation(ctx context.Context, claim *v1.PersistentVolumeClaim)
	....
	volume, result, err := ctrl.provisioner.Provision(ctx, options)
	....
```
