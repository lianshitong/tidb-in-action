## 1.4.1 基于 TiOps 的集群滚动更新

滚动升级功能借助 TiDB 的分布式能力，升级过程中尽量保证对前端业务透明、无感知。升级时会先检查各个组件的配置文件是否合理，如果配置有问题，则报错退出；如果配置没有问题，则工具会逐个节点升级。其中对不同节点有不同的操作。

### 1.4.1.1 不同节点的操作
+ 升级 PD
    - 优先升级非 Leader 节点
    - 所有非 Leader 节点升级完成后再升级 Leader 节点
        - 工具会向 PD 发送一条命令将 Leader 迁移到升级完成的节点上
        - 当 Leader 已经切换到其他节点之后，再对旧的 Leader 节点做升级操作
    - 同时升级过程中，若发现有不健康的节点时工具会中止本次升级并退出，此时需要由人工判断、修复后再执行升级。
- 升级 TiKV
    - 先在 PD 中添加一个迁移对应 TiKV 上 region leader 的调度，通过迁移 Leader 确保升级过程中不影响前端业务。
    - 等待迁移 Leader 完成之后，再对该 TiKV 节点进行升级更新
    - 等更新后的 TiKV 正常启动之后再移除迁移 Leader 的调度
- 升级其他服务
    - 正常停止服务更新

### 1.4.1.2 版本升级操作

上面提到的逻辑都是在工具中集成好的功能，以升级到 4.0.0-beta.1 版本为例，我们实际上只需要一条命令就可以实现整个集群的版本升级。

```
$ tiops upgrade -c tidb-test -t 4.0.0-beta.1
```

```
-c | --cluster-name 必选参数。用以标识需要扩容的集群
-t | --tidb-version 必选参数。用于表示升级的目标版本
--enable-check-config：可选参数。检查配置文件是否合法，默认：disable
--local-pkg 可选参数。若无外网，可将安装包拷贝中控机本地，通过此参数指相关路径进行离线安装
-f | --forks 可选参数。执行命令时的并发数，默认：5；当节点数比较多时，可以适当调大
-r | --role 可选参数。对指定服务进行升级操作，role 按照 TiDB 服务的角色类型取值："pd", "tikv", "pump", "tidb",  "drainer", "monitoring", "monitored", "grafana", "alertmanager"
-n | --node-id 可选参数。指定升级的节点 ID
--force 可算参数。常规情况是滚动升级，设置此参数，升级时会强制停机、重启
```
