# Volcano 测试傻瓜文档

这份文档面向“刚完成一个功能或修复一个 bug，不知道该怎么测”的场景。目标是让你按改动范围选择合适的测试，先快速验证，再补充必要的 e2e，最后带着清晰的测试结果提交代码。

## 1. 先判断你改了什么

先看自己的改动范围：

```bash
git status --short
git diff --stat
```

然后按下面表格选择测试：

| 改动类型 | 必跑测试 | 建议补充 |
| --- | --- | --- |
| 只改文档 | 不需要跑单测；检查 Markdown 是否清晰 | 如改了示例命令，可手动验证命令 |
| 改 `pkg/` 下普通函数、工具方法、插件逻辑 | 对应包单测 + `make unit-test` | `make verify` |
| 改 Scheduler Action/Plugin/Cache/Session | 对应包单测 + `make unit-test` | `make e2e-test-schedulingbase` 或 `make e2e-test-schedulingaction` |
| 改 Job/PodGroup/Queue/CronJob/JobFlow Controller | 对应包单测 + `make unit-test` | `make e2e-test-jobp`、`make e2e-test-jobseq`、`make e2e-test-cronjob` |
| 改 Admission Webhook | 对应包单测 + `make unit-test` | `make e2e-test-admission-webhook` 或 `make e2e-test-admission-policy` |
| 改 CLI/vcctl | CLI 相关单测 + `make vcctl` | `make e2e-test-vcctl` |
| 改 CRD/API 类型、生成代码、YAML、Helm | `make verify` + `make generate-yaml` | `make verify-generated-yaml`，必要时跑相关 e2e |
| 改 HyperNode/网络拓扑调度 | 对应包单测 + `make unit-test` | `make e2e-test-hypernode` |
| 改 DRA、设备、GPU/NPU 共享 | 对应包单测 + `make unit-test` | `make e2e-test-dra` 或相关调度 e2e |
| 改 Agent/Agent-Scheduler/混部 | 对应包单测 + 构建对应二进制 | 有条件再跑集群级手动验证 |

最低要求：功能代码或 bugfix 至少跑相关包单测和 `make unit-test`。影响 CRD、生成代码、格式或公共接口时，再跑 `make verify`。

## 2. 本地环境准备

基础依赖：

- Go：以 `go.mod` 为准，当前项目声明 `go 1.25.0`。
- Docker：跑 e2e 和构建镜像需要。
- kind：`hack/run-e2e-kind.sh` 会用 kind 创建临时 Kubernetes 集群。
- Helm、kubectl：e2e 脚本会安装 Volcano Chart 并操作集群。
- 网络可访问 Go module、容器镜像和 Helm 依赖。

先确认命令可用：

```bash
go version
docker version
kind version
kubectl version --client
helm version
```

如果只跑单元测试，通常只需要 Go 环境。

## 3. 最常用测试命令

### 3.1 跑单个包测试

当你只改了一个包，优先跑这个包，速度最快：

```bash
go test ./pkg/scheduler/plugins/gang
```

如果要看详细输出：

```bash
go test -v ./pkg/scheduler/plugins/gang
```

如果只想跑某一个测试函数：

```bash
go test -v ./pkg/scheduler/plugins/gang -run TestName
```

如果这个包有并发或状态问题，建议加 race：

```bash
go test -race ./pkg/scheduler/plugins/gang
```

### 3.2 跑全部单元测试

项目提供了统一入口：

```bash
make unit-test
```

Linux 下这个目标会清理测试缓存，并对 `pkg`、`cmd` 中包含 `*_test.go` 的包执行 `go test -race`。

### 3.3 跑格式、生成代码校验

提交前建议跑：

```bash
make verify
```

它会执行：

- `hack/verify-gofmt.sh`
- `hack/verify-gencode.sh`

如果你改了 CRD、Helm、安装 YAML 或生成产物，再跑：

```bash
make generate-yaml
make verify-generated-yaml
```

如果你改了 Go 代码风格或想提前发现 lint 问题：

```bash
make lint
```

## 4. e2e 测试怎么跑

Volcano 的 e2e 测试在 `test/e2e/` 下，使用 Ginkgo。项目推荐通过 Makefile 启动，它会先构建镜像，再用 kind 创建临时集群，安装 Volcano，然后运行指定 e2e 套件。

### 4.1 跑全部 e2e

```bash
make e2e
```

这个命令耗时较长，适合大改动或提交前完整验证。

### 4.2 按模块跑 e2e

常用目标如下：

| 命令 | 对应场景 |
| --- | --- |
| `make e2e-test-schedulingbase` | 基础调度能力，如 DRF、SLA、VolumeBinding、基础作业调度 |
| `make e2e-test-schedulingaction` | 调度 Action，如 enqueue、allocate、preempt、reclaim、guarantee |
| `make e2e-test-jobp` | Job 生命周期、PodGroup Controller、minSuccess、扩缩容、重启策略 |
| `make e2e-test-jobseq` | Job 插件和分布式框架，如 TensorFlow、PyTorch、MPI、Ray |
| `make e2e-test-cronjob` | Volcano CronJob |
| `make e2e-test-vcctl` | vcctl 命令行 |
| `make e2e-test-admission-webhook` | Admission Webhook mutate/validate |
| `make e2e-test-admission-policy` | ValidatingAdmissionPolicy/MutatingAdmissionPolicy |
| `make e2e-test-hypernode` | HyperNode 与网络拓扑感知 |
| `make e2e-test-dra` | Dynamic Resource Allocation |
| `make e2e-test-stress` | 压力测试 |

例子：你改了 `pkg/scheduler/actions/preempt`，推荐：

```bash
go test ./pkg/scheduler/actions/preempt
make unit-test
make e2e-test-schedulingaction
```

例子：你改了 `pkg/webhooks`：

```bash
go test ./pkg/webhooks/...
make unit-test
make e2e-test-admission-webhook
```

### 4.3 e2e 常用环境变量

`hack/run-e2e-kind.sh` 支持一些环境变量：

| 变量 | 作用 | 默认值 |
| --- | --- | --- |
| `CLUSTER_NAME` | kind 集群名 | `integration` |
| `CLEANUP_CLUSTER` | 测试结束后是否删除 kind 集群，`1` 删除，`0` 保留 | `1` |
| `ARTIFACTS_PATH` | e2e 失败时导出的日志目录 | `volcano-e2e-logs` |
| `IMAGE_PREFIX` | 构建镜像前缀 | `volcanosh` |
| `TAG` | 镜像 tag，默认使用当前 git commit | 当前 commit |
| `E2E_TYPE` | e2e 套件类型，一般由 Makefile 目标设置 | `ALL` |
| `FEATURE_GATES` | 安装 Volcano 时传入的 feature gates | 空 |
| `IGNORED_PROVISIONERS` | 调度器忽略的 CSI provisioner | 空 |

保留失败现场方便排查：

```bash
CLEANUP_CLUSTER=0 make e2e-test-schedulingaction
```

查看集群：

```bash
kubectl get pods -n volcano-system
kubectl get pods -A
```

排查完成后手动删除 kind 集群：

```bash
kind delete cluster --name integration
```

指定日志目录：

```bash
ARTIFACTS_PATH=/tmp/volcano-e2e-logs make e2e-test-jobp
```

## 5. 在已有集群上跑 e2e

如果你已经有一个 Kubernetes 集群，并且已经部署了当前代码对应的 Volcano，可以直接跑某个 e2e 包：

```bash
make vcctl
export VC_BIN=$(pwd)/_output/bin
KUBECONFIG=${KUBECONFIG} go test ./test/e2e/jobp
```

如果使用 Ginkgo：

```bash
go install github.com/onsi/ginkgo/v2/ginkgo@latest
export VC_BIN=$(pwd)/_output/bin
KUBECONFIG=${KUBECONFIG} ginkgo -v -r ./test/e2e/jobp
```

注意：已有集群方式不会自动帮你构建镜像、安装 Chart 或清理环境。你需要自己保证集群里运行的 Volcano 镜像就是当前改动生成的镜像。

## 6. 新功能应该怎么补测试

### 6.1 先补单元测试

单元测试文件放在被测包旁边，命名为 `xxx_test.go`。Volcano 的单元测试使用 Go 标准 `testing` 包，也会使用 `testify`、`gomega` 等辅助库。

推荐使用表驱动：

```go
func TestSomething(t *testing.T) {
	tests := []struct {
		name string
		input string
		want string
	}{
		{name: "empty input", input: "", want: "default"},
		{name: "normal input", input: "demo", want: "demo"},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := Something(tt.input)
			if got != tt.want {
				t.Fatalf("got %q, want %q", got, tt.want)
			}
		})
	}
}
```

单元测试应该尽量 hermetic：不要依赖真实集群、真实网络、真实时间或测试顺序。需要 Kubernetes 对象时，优先使用 fake client、构造 API 对象或已有 test util。

### 6.2 再判断是否需要 e2e

需要补 e2e 的典型情况：

- 用户可见行为变化，例如 Job 状态、PodGroup 状态、Queue 状态变化。
- Scheduler 决策变化，例如某个 Pod 应该调度到哪个节点、是否抢占、是否回收。
- Admission 行为变化，例如创建资源时应该被 mutate 或 validate 拒绝。
- Controller 会创建、删除或更新 Kubernetes 资源。
- 修复的是线上 bug，单测无法覆盖完整控制面交互。

不一定需要 e2e 的情况：

- 纯重构，行为不变。
- 私有工具函数修复，单测能覆盖输入输出。
- 只改日志、注释、文档。

### 6.3 e2e 放在哪里

按功能放入对应目录：

| 目录 | 适合放的测试 |
| --- | --- |
| `test/e2e/schedulingbase` | 基础调度策略和普通调度能力 |
| `test/e2e/schedulingaction` | allocate、enqueue、preempt、reclaim 等 Action 行为 |
| `test/e2e/jobp` | Job Controller、PodGroup、Job 生命周期 |
| `test/e2e/jobseq` | Job 插件、分布式框架、任务序列 |
| `test/e2e/admission` | Webhook、AdmissionPolicy |
| `test/e2e/cronjob` | CronJob |
| `test/e2e/vcctl` | CLI |
| `test/e2e/hypernode` | HyperNode、网络拓扑 |
| `test/e2e/dra` | DRA |
| `test/e2e/stress` | 压测 |

每个 e2e 包都有 `main_test.go` 或 `e2e_test.go` 负责 `RunSpecs`。新增 case 时通常只需要在对应包中新增或修改 `Describe/Context/It`。

## 7. bugfix 测试套路

修 bug 时按这个顺序做：

1. 先复现：写一个失败的单测或 e2e，证明 bug 存在。
2. 再修复：改实现，让刚才的测试通过。
3. 跑相关包测试：确认修复点附近没有破坏。
4. 跑 `make unit-test`：确认基础回归。
5. 如果 bug 涉及真实 Kubernetes 交互，跑对应 e2e。
6. 在 PR 或提交说明里写清楚跑过哪些测试。

示例：修复 Queue 关闭时仍允许创建 Job 的问题：

```bash
go test ./pkg/webhooks/...
go test ./pkg/controllers/queue
make unit-test
make e2e-test-admission-webhook
```

## 8. feature 测试套路

做新功能时按这个顺序：

1. 写单元测试覆盖核心算法、边界条件和错误分支。
2. 如果有 API 字段变更，跑生成代码和 YAML：

```bash
make generate-code
make generate-yaml
make verify
make verify-generated-yaml
```

3. 构建受影响组件：

```bash
make vc-scheduler
make vc-controller-manager
make vc-webhook-manager
```

按实际改动选择，不需要每次都构建所有组件。

4. 跑相关 e2e。
5. 如功能有用户操作入口，补充手动验证步骤或示例 YAML。

## 9. 常见模块测试清单

### Scheduler Plugin

```bash
go test ./pkg/scheduler/plugins/<plugin-name>
make unit-test
make e2e-test-schedulingbase
```

如果插件影响抢占或回收：

```bash
make e2e-test-schedulingaction
```

### Scheduler Action

```bash
go test ./pkg/scheduler/actions/<action-name>
make unit-test
make e2e-test-schedulingaction
```

### Controller

```bash
go test ./pkg/controllers/<controller-name>
make unit-test
```

根据控制器类型选择：

```bash
make e2e-test-jobp
make e2e-test-jobseq
make e2e-test-cronjob
```

### Admission

```bash
go test ./pkg/webhooks/...
make unit-test
make e2e-test-admission-webhook
```

如果改的是 VAP/MAP：

```bash
make e2e-test-admission-policy
```

### CLI

```bash
go test ./pkg/cli/...
make vcctl
make e2e-test-vcctl
```

### API / CRD

```bash
make generate-code
make generate-yaml
make verify
make verify-generated-yaml
make unit-test
```

### HyperNode / 拓扑

```bash
go test ./pkg/scheduler/plugins/network-topology-aware
go test ./pkg/controllers/hypernode/...
make unit-test
make e2e-test-hypernode
```

## 10. 测试失败时看哪里

### 单元测试失败

先只跑失败包：

```bash
go test -v ./path/to/package -run TestName
```

清理缓存再跑：

```bash
go clean -testcache
go test -v ./path/to/package
```

常见原因：

- 测试依赖 map 遍历顺序，导致随机失败。
- 没有重置全局变量或 fake client 状态。
- 时间相关测试没有使用可控 clock。
- race 测试失败，说明存在并发读写问题。

### e2e 失败

默认失败时脚本会导出 kind 日志到 `volcano-e2e-logs`。如果想保留集群，使用：

```bash
CLEANUP_CLUSTER=0 make e2e-test-schedulingaction
```

然后查看：

```bash
kubectl get pods -n volcano-system
kubectl logs -n volcano-system deploy/volcano-scheduler
kubectl logs -n volcano-system deploy/volcano-controllers
kubectl logs -n volcano-system deploy/volcano-admission
kubectl get podgroups -A
kubectl get queues
kubectl describe pod <pod-name> -n <namespace>
```

常见原因：

- 本地镜像没构建成功，集群跑的不是当前代码。
- kind 集群资源不足。
- Webhook 没 ready，资源创建被拒绝或超时。
- 测试依赖异步状态，等待时间不够。
- 上一次保留的 kind 集群里有残留资源。

清理后重试：

```bash
kind delete cluster --name integration
make e2e-test-schedulingaction
```

## 11. 本地和远程调试

测试能告诉你“坏了”，调试要解决“为什么坏”。Volcano 的调试分三种情况：完全没有 Kubernetes 集群、用本机 kind 临时集群、连接远程 Kubernetes 集群。

### 11.1 没有 K8s 集群时怎么本地调试

没有任何 Kubernetes 集群时，只能调试不依赖真实 API Server 的逻辑。适合调试算法、状态转换、插件计算、Webhook 校验函数、Controller reconcile 内部函数等。

优先使用这几种方式：

| 调试方式 | 适合场景 |
| --- | --- |
| 单包 `go test` | 调试普通函数、调度插件、Action、Controller 内部逻辑 |
| fake client 单测 | 调试 Kubernetes 对象读写逻辑，但不需要真实 API Server |
| `httptest` | 调试 extender、webhook handler、HTTP 相关逻辑 |
| 日志 + 表驱动测试 | 快速定位输入输出不符合预期的分支 |

最常用命令：

```bash
go test -v ./pkg/scheduler/plugins/gang -run TestName
go test -v ./pkg/controllers/job -run TestName
go test -v ./pkg/webhooks/... -run TestName
```

需要单步调试时，可以用 Delve：

```bash
go install github.com/go-delve/delve/cmd/dlv@latest
dlv test ./pkg/scheduler/plugins/gang -- -test.run TestName
```

进入 Delve 后常用命令：

```text
b TestName          # 给测试函数打断点
b file.go:123       # 给某一行打断点
c                   # continue
n                   # next
s                   # step
p variableName      # 打印变量
bt                  # 查看调用栈
q                   # 退出
```

如果你改的是调度算法，建议构造 `JobInfo`、`TaskInfo`、`NodeInfo` 等对象做单元测试，而不是一开始就跑 e2e。这样反馈最快，也更容易稳定复现边界条件。

### 11.2 没有现成集群，但可以在本机创建 kind

如果你本机没有现成 Kubernetes 集群，但装了 Docker、kind、kubectl、Helm，可以让项目脚本自动创建一个临时集群。

最省事的 e2e 调试方式：

```bash
CLEANUP_CLUSTER=0 make e2e-test-schedulingaction
```

`CLEANUP_CLUSTER=0` 会保留 kind 集群，方便失败后继续查看现场。默认集群名是 `integration`。

查看 Volcano 组件：

```bash
kubectl get pods -n volcano-system
kubectl logs -n volcano-system deploy/volcano-scheduler
kubectl logs -n volcano-system deploy/volcano-controllers
kubectl logs -n volcano-system deploy/volcano-admission
```

查看调度资源：

```bash
kubectl get queues
kubectl get podgroups -A
kubectl get vcjobs -A
kubectl get pods -A
```

调试完成后清理：

```bash
kind delete cluster --name integration
```

也可以使用项目的一键本地启动脚本：

```bash
./hack/local-up-volcano.sh
```

这个脚本会构建镜像、创建 kind 集群并安装 Volcano。默认集群名是 `volcano`。清理方式：

```bash
./hack/local-up-volcano.sh -q
```

保留 kind 集群后，你可以把某个组件停掉，在本机直接运行二进制连接这个 kind 集群。以 Scheduler 为例：

```bash
kubectl scale deploy/volcano-scheduler -n volcano-system --replicas=0
make vc-scheduler
./_output/bin/vc-scheduler \
  --kubeconfig=$HOME/.kube/config \
  --scheduler-conf=$(pwd)/installer/helm/chart/volcano/config/volcano-scheduler.conf \
  --enable-healthz=true \
  --enable-metrics=true \
  -v=5
```

这样适合用 IDE 或 Delve 给本地进程打断点：

```bash
dlv exec ./_output/bin/vc-scheduler -- \
  --kubeconfig=$HOME/.kube/config \
  --scheduler-conf=$(pwd)/installer/helm/chart/volcano/config/volcano-scheduler.conf \
  -v=5
```

Controller Manager 也可以类似处理：

```bash
kubectl scale deploy/volcano-controllers -n volcano-system --replicas=0
make vc-controller-manager
./_output/bin/vc-controller-manager \
  --kubeconfig=$HOME/.kube/config \
  --enable-healthz=true \
  --enable-metrics=true \
  -v=5
```

注意：本机直接运行组件时，要先停掉集群里的同名组件，否则两个实例会同时 watch 和更新资源，导致结果混乱。

### 11.3 K8s 集群在远程时怎么调试

远程集群调试有两条路线：把当前代码构建成镜像部署到远程集群，或者把远程集群的 kubeconfig 拿到本地，让本地进程连接远程 API Server。

#### 路线 A：构建镜像，部署到远程集群

适合调试 Webhook、Controller、Scheduler 的真实部署行为，也适合多人共享测试环境。

1. 确认 kubeconfig 指向远程集群：

```bash
export KUBECONFIG=/path/to/remote.kubeconfig
kubectl cluster-info
kubectl get nodes
```

2. 设置镜像仓库和 tag：

```bash
export IMAGE_PREFIX=<your-registry-or-namespace>
export TAG=debug-$(date +%Y%m%d%H%M%S)
```

3. 构建并推送镜像。不同 Docker 环境参数可能不同，常见方式是：

```bash
make images BUILDX_OUTPUT_TYPE=registry IMAGE_PREFIX=${IMAGE_PREFIX} TAG=${TAG}
```

如果你的 buildx 不直接推送，可以先本地构建，再按自己的镜像仓库流程 `docker push`。

4. 用 Helm 安装或升级远程集群里的 Volcano：

```bash
helm upgrade --install volcano installer/helm/chart/volcano \
  -n volcano-system \
  --create-namespace \
  --set basic.image_registry=${IMAGE_PREFIX} \
  --set basic.image_tag_version=${TAG} \
  --set basic.image_pull_policy=Always
```

如果 Chart 的镜像字段和当前分支有差异，先查看 values：

```bash
helm show values installer/helm/chart/volcano
```

5. 查看组件是否使用新镜像：

```bash
kubectl get pods -n volcano-system
kubectl describe pod -n volcano-system -l app=volcano-scheduler
kubectl logs -n volcano-system deploy/volcano-scheduler
```

6. 跑远程集群上的 e2e：

```bash
make vcctl
export VC_BIN=$(pwd)/_output/bin
KUBECONFIG=${KUBECONFIG} ginkgo -v -r ./test/e2e/jobp
```

如果不想装 Ginkgo，也可以用：

```bash
KUBECONFIG=${KUBECONFIG} go test ./test/e2e/jobp
```

#### 路线 B：本地运行组件，连接远程 API Server

适合单人调试 Scheduler 或 Controller 逻辑，可以在本机 IDE/Delve 中打断点。前提是远程 API Server 能从本机访问，并且 kubeconfig 有足够权限。

以远程 Scheduler 调试为例：

```bash
export KUBECONFIG=/path/to/remote.kubeconfig
kubectl scale deploy/volcano-scheduler -n volcano-system --replicas=0
make vc-scheduler
dlv exec ./_output/bin/vc-scheduler -- \
  --kubeconfig=${KUBECONFIG} \
  --scheduler-conf=$(pwd)/installer/helm/chart/volcano/config/volcano-scheduler.conf \
  -v=5
```

以远程 Controller Manager 调试为例：

```bash
export KUBECONFIG=/path/to/remote.kubeconfig
kubectl scale deploy/volcano-controllers -n volcano-system --replicas=0
make vc-controller-manager
dlv exec ./_output/bin/vc-controller-manager -- \
  --kubeconfig=${KUBECONFIG} \
  -v=5
```

调试结束后恢复远程组件：

```bash
kubectl scale deploy/volcano-scheduler -n volcano-system --replicas=1
kubectl scale deploy/volcano-controllers -n volcano-system --replicas=1
```

这种方式的关键点：

- 一次只本地运行一个同类组件，避免和集群内组件抢资源或重复 reconcile。
- 不要在共享集群上长时间 scale down 公共组件。
- 调试前记录原始副本数，结束后恢复。
- 远程网络延迟会影响 watch 和调度反馈，日志级别建议使用 `-v=5` 或更高临时排查。

#### Webhook 远程调试特别注意

Webhook 和 Scheduler/Controller 不一样。Kubernetes API Server 需要主动访问 Webhook 服务，所以“本地启动 webhook 进程”通常不能直接被远程 API Server 访问。

推荐做法：

1. 优先用单元测试调试 webhook handler 和 validate/mutate 逻辑。
2. 需要真实准入链路时，构建 webhook 镜像并部署到远程集群。
3. 如果必须本地断点调试，需要准备 API Server 可访问的公网或内网回调地址，并用 `--webhook-url` 或对应 WebhookConfiguration 指向该地址。这种方式对网络、证书和安全要求更高，不建议作为日常首选。

Webhook 常用验证命令：

```bash
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
kubectl logs -n volcano-system deploy/volcano-admission
```

### 11.4 本地进程调试后的恢复清单

调试完务必恢复环境：

```bash
kubectl get deploy -n volcano-system
kubectl scale deploy/volcano-scheduler -n volcano-system --replicas=1
kubectl scale deploy/volcano-controllers -n volcano-system --replicas=1
kubectl scale deploy/volcano-admission -n volcano-system --replicas=1
kubectl get pods -n volcano-system
```

如果是 kind 临时集群：

```bash
kind delete cluster --name integration
kind delete cluster --name volcano
```

如果是远程共享集群，确认没有遗留测试资源：

```bash
kubectl get vcjobs -A
kubectl get podgroups -A
kubectl get queues
```

## 12. 提交前最小检查

如果你不知道该跑什么，按这个最小集合：

```bash
make unit-test
make verify
```

如果改了调度行为，再加一个：

```bash
make e2e-test-schedulingbase
```

如果改了控制器行为，再加一个：

```bash
make e2e-test-jobp
```

如果改了准入行为，再加一个：

```bash
make e2e-test-admission-webhook
```

提交 PR 时建议写清楚：

```text
Tested:
- go test ./pkg/scheduler/plugins/gang
- make unit-test
- make e2e-test-schedulingbase
```

## 13. 一句话原则

小改动先跑相关包单测，大改动跑 `make unit-test` 和 `make verify`，用户可见行为变化一定补对应 e2e。测试不是越多越好，而是要覆盖你改动可能破坏的行为。
