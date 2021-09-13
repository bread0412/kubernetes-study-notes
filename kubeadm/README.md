# 本次主題：kubeadm原始碼分析

## 原始碼目錄
目錄名 | 用途
------------- | -------------
cmd | 每個組件的的入口(main函數)，透過cobra cli工具初始化，參數跟文件也是透過cobra來進行解析
pkg | 每個組件的具體功能
staging | 此目錄下的元件，其實是k8s.io下單獨的項目，有獨立自己存放庫的元件，程式碼更改還在kubernetes的git下，定時merge到對應的項目
vendor | 依賴第三方

## kubeadm 流程
- 延伸補出：建議可閱讀一下 cobra

入口
```go
// cmd/kubeadm/app/kubeadm.go
// Run creates and executes new kubeadm command
func Run() error {
	klog.InitFlags(nil)
	pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	pflag.CommandLine.AddGoFlagSet(flag.CommandLine)

	pflag.Set("logtostderr", "true")
	// We do not want these flags to show up in --help
	// These MarkHidden calls must be after the lines above
	pflag.CommandLine.MarkHidden("version")
	pflag.CommandLine.MarkHidden("log-flush-frequency")
	pflag.CommandLine.MarkHidden("alsologtostderr")
	pflag.CommandLine.MarkHidden("log-backtrace-at")
	pflag.CommandLine.MarkHidden("log-dir")
	pflag.CommandLine.MarkHidden("logtostderr")
	pflag.CommandLine.MarkHidden("stderrthreshold")
	pflag.CommandLine.MarkHidden("vmodule")

	cmd := cmd.NewKubeadmCommand(os.Stdin, os.Stdout, os.Stderr)
	return cmd.Execute()
}
```
這邊可以看到kubeadm有支援什麼command，比較常用的有
- kubeadm init
- kubeadm join
- kubeadm token
- kubeadm config
- kubeadm version
> 其中kubeadm init和join兩個命令最重要，在 master 執行 kubeadm init，完成 master 的初始化。接著再在其他 node 上執行 kubeadm join 將 node 加到 cluster，就完成一個群集！

```go
// cmd/kubeadm/app/cmd/cmd.go
// NewKubeadmCommand returns cobra.Command to run kubeadm command
func NewKubeadmCommand(in io.Reader, out, err io.Writer) *cobra.Command {
    
    // 省略中間程式碼

	cmds.AddCommand(newCmdCertsUtility(out))
	cmds.AddCommand(newCmdCompletion(out, ""))
	cmds.AddCommand(newCmdConfig(out))
	cmds.AddCommand(newCmdInit(out, nil))
	cmds.AddCommand(newCmdJoin(out, nil))
	cmds.AddCommand(newCmdReset(in, out, nil))
	cmds.AddCommand(newCmdVersion(out))
	cmds.AddCommand(newCmdToken(out, err))
	cmds.AddCommand(upgrade.NewCmdUpgrade(out))
	cmds.AddCommand(alpha.NewCmdAlpha())
	options.AddKubeadmOtherFlags(cmds.PersistentFlags(), &rootfsPath)
	cmds.AddCommand(newCmdKubeConfigUtility(out))

	return cmds
}
```
### kubeadm init 工作機制
- 主要工作
    - 創建群集相關的 key, certs, conf 等文件
    - 創建 kube-apiserver, kube-controller-manager, kube-scheduler, etcd 這些 static pod 的 json manifest 文件，kubelet 負責啟動這些 master 組建
    - 通過 addons 方式啟動 kube-discovery deployment, kube-proxy daemonset, kube-dns deployment

```go
// cmd/kubeadm/app/cmd/init.go
// newCmdInit returns "kubeadm init" command.
// NB. initOptions is exposed as parameter for allowing unit testing of
//     the newInitOptions method, that implements all the command options validation logic
func newCmdInit(out io.Writer, initOptions *initOptions) *cobra.Command {

    // 省略中間
    // 主要就是做下面這幾件事情
    // initialize the workflow runner with the list of phases
	initRunner.AppendPhase(phases.NewPreflightPhase())
	initRunner.AppendPhase(phases.NewCertsPhase())
	initRunner.AppendPhase(phases.NewKubeConfigPhase())
	initRunner.AppendPhase(phases.NewKubeletStartPhase())
	initRunner.AppendPhase(phases.NewControlPlanePhase())
	initRunner.AppendPhase(phases.NewEtcdPhase())
	initRunner.AppendPhase(phases.NewWaitControlPlanePhase())
	initRunner.AppendPhase(phases.NewUploadConfigPhase())
	initRunner.AppendPhase(phases.NewUploadCertsPhase())
	initRunner.AppendPhase(phases.NewMarkControlPlanePhase())
	initRunner.AppendPhase(phases.NewBootstrapTokenPhase())
	initRunner.AppendPhase(phases.NewKubeletFinalizePhase())
    initRunner.AppendPhase(phases.NewAddonPhase())
    
```
![init workflow(本圖ref https://cloud.tencent.com/developer/article/1096515)]()


### kubeadm join 工作機制
- 主要工作
    - 訪問 kube-discovery 服務，獲取cluster info (包含 ca 證書、api server endpoint 列表和 token)
    - 利用 token 檢驗 cluster info
    - 檢驗成功後，再與 api server 建立連接，請求 api server 為該 node 創建證書
    - 根據獲取到的證書創建 kubelet.conf

![join workflow(本圖ref https://cloud.tencent.com/developer/article/1096515)]()



## 補充資料
* [kubernetes kubeadm 官方](https://kubernetes.io/docs/reference/setup-tools/kubeadm/#config-file)

  