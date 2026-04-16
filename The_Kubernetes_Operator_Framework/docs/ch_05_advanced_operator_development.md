# Developing an Operator – Advanced Functionality

## Understanding the need for advanced functionality

### Kiến thức cần nhớ

* Một Operator chỉ có thể cài Operand và quản lý cơ bản đã là **có giá trị**.
* Không nhất thiết mọi Operator đều phải có toàn bộ tính năng nâng cao.
* Tuy nhiên, khi đi gần production hơn, người dùng sẽ cần:

  * **insight chi tiết hơn** về hành vi của Operator
  * **khả năng debug failure state dễ hơn**
  * **observability tốt hơn** qua metrics và status conditions 

### Ý chính cần ghi nhớ

* Dừng ở Level I/II là hoàn toàn chấp nhận được.
* Nhưng nếu muốn tiến lên các mức capability cao hơn, bạn thường phải thêm:

  * conditions
  * metrics
  * leader election
  * health/readiness checks

### Note học nhanh

* **Bắt đầu đơn giản** là hợp lý.
* **Thêm advanced functionality khi có nhu cầu vận hành thật**.

---

## Reporting status conditions

### Kiến thức lý thuyết cần nhớ

* `status.conditions` là nơi chuẩn trong Kubernetes để biểu diễn **trạng thái hiện tại** của resource.
* So với logs, conditions có lợi thế:

  * nằm ngay trong CR
  * dễ xem bằng `kubectl describe`
  * có cấu trúc rõ ràng cho cả người và máy
* Operator Framework và OLM đều tận dụng mô hình condition chuẩn của Kubernetes.

### Ý chính cần nhớ

* Có hai hướng báo condition trong chương này:

  1. ghi condition vào **CRD status** của chính Operator
  2. ghi condition vào **OperatorCondition** để OLM đọc 

---

## Operator CRD conditions

### Kiến thức cần nhớ

* CR của Operator nên có cả:

  * `spec`: đầu vào cấu hình
  * `status`: đầu ra trạng thái
* `status.conditions` được dùng để báo tình trạng mới nhất của Operator. 

### Code sách sử dụng

```go
// NginxOperatorStatus defines the observed state of NginxOperator
type NginxOperatorStatus struct {

   // Conditions is the list of status condition updates

   Conditions []metav1.Condition `json:"conditions"`

}
```

Sau đó chạy:

```bash
make generate
```

```bash
make manifests
```

hoặc:

```bash
make
```



### Giải thích chi tiết code

* `NginxOperatorStatus` là struct đại diện cho phần `status` của CR.
* `Conditions []metav1.Condition`:

  * dùng kiểu condition chuẩn của Kubernetes
  * mỗi phần tử sẽ có `Type`, `Status`, `Reason`, `Message`, `LastTransitionTime`
* `json:"conditions"`:

  * field sẽ xuất hiện trong YAML/JSON dưới dạng `status.conditions`

### Giải thích chi tiết command

* `make generate`

  * regenerate code phụ thuộc vào API/type definitions
  * cập nhật các file generated như deepcopy, client code
* `make manifests`

  * regenerate CRD manifests từ Go types và markers
* `make`

  * thường chạy trọn bộ các bước generate/build targets trong project scaffold

### Code cập nhật `Reconcile()` để set condition

```go
func (r *NginxOperatorReconciler) Reconcile(ctx context.
Context, req ctrl.Request) (ctrl.Result, error) {

   logger := log.FromContext(ctx)

   operatorCR := &operatorv1alpha1.NginxOperator{}

   err := r.Get(ctx, req.NamespacedName, operatorCR)

   if err != nil && errors.IsNotFound(err) {

      logger.Info("Operator resource object not found.")

      return ctrl.Result{}, nil

   } else if err != nil {

      logger.Error(err, "Error getting operator resource 
object")

      meta.SetStatusCondition(&operatorCR.Status.Conditions, 
metav1.Condition{

         Type:               "OperatorDegraded",

         Status:             metav1.ConditionTrue,

         Reason:             "OperatorResourceNotAvailable",

         LastTransitionTime: metav1.NewTime(time.Now()),

         Message:            fmt.Sprintf("unable to get 
operator custom resource: %s", err.Error()),

      })

      return ctrl.Result{}, utilerrors.NewAggregate([]
error{err, r.Status().Update(ctx, operatorCR)})

   }

   deployment := &appsv1.Deployment{}

   create := false

   err = r.Get(ctx, req.NamespacedName, deployment)

   if err != nil && errors.IsNotFound(err) {

      create = true

      deployment = assets.GetDeploymentFromFile("assets/nginx_
deployment.yaml")

   } else if err != nil {

      logger.Error(err, "Error getting existing Nginx 
deployment.")

      meta.SetStatusCondition(&operatorCR.Status.Conditions, 
metav1.Condition{

         Type:               "OperatorDegraded",

         Status:             metav1.ConditionTrue,

         Reason:             "OperandDeploymentNotAvailable",

         LastTransitionTime: metav1.NewTime(time.Now()),

         Message:            fmt.Sprintf("unable to get operand 
deployment: %s", err.Error()),

      })

      return ctrl.Result{}, utilerrors.NewAggregate([]
error{err, r.Status().Update(ctx, operatorCR)})

   }

   deployment.Namespace = req.Namespace

   deployment.Name = req.Name

   if operatorCR.Spec.Replicas != nil {

      deployment.Spec.Replicas = operatorCR.Spec.Replicas

   }

   if operatorCR.Spec.Port != nil {

      deployment.Spec.Template.Spec.Containers[0].Ports[0].
ContainerPort = *operatorCR.Spec.Port

   }

   ctrl.SetControllerReference(operatorCR, deployment, 
r.Scheme)

   if create {

      err = r.Create(ctx, deployment)

   } else {

      err = r.Update(ctx, deployment)

   }

if err != nil {

   meta.SetStatusCondition(&operatorCR.Status.Conditions, 
metav1.Condition{

      Type:               "OperatorDegraded",

      Status:             metav1.ConditionTrue,

      Reason:             "OperandDeploymentFailed",

      LastTransitionTime: metav1.NewTime(time.Now()),

      Message:            fmt.Sprintf("unable to update operand 
deployment: %s", err.Error()),

   })

   return ctrl.Result{}, utilerrors.NewAggregate([]error{err, 
r.Status().Update(ctx, operatorCR)})

}

   meta.SetStatusCondition(&operatorCR.Status.Conditions, 
metav1.Condition{

      Type:               "OperatorDegraded",

      Status:             metav1.ConditionFalse,

      Reason:             "OperatorSucceeded",

      LastTransitionTime: metav1.NewTime(time.Now()),

      Message:            "operator successfully reconciling",

   })

   return ctrl.Result{}, utilerrors.NewAggregate([]error{err, 
r.Status().Update(ctx, operatorCR)})

}
```

### Giải thích chi tiết code

* `meta.SetStatusCondition(...)`

  * helper để thêm/cập nhật condition trong `status.conditions`
* `Type: "OperatorDegraded"`

  * condition chính mà sách dùng
* `Status: metav1.ConditionTrue`

  * nghĩa là Operator đang degraded
* `Status: metav1.ConditionFalse`

  * nghĩa là Operator không degraded
* `Reason`

  * lý do ngắn gọn, ổn định, machine-friendly
* `Message`

  * diễn giải dễ hiểu cho người đọc
* `LastTransitionTime`

  * mốc thời gian đổi trạng thái
* `r.Status().Update(ctx, operatorCR)`

  * update riêng phần status của CR
* `utilerrors.NewAggregate(...)`

  * gộp nhiều lỗi thành một lỗi trả về

### Các reason sách dùng

* `OperatorResourceNotAvailable`
* `OperandDeploymentNotAvailable`
* `OperandDeploymentFailed`
* `OperatorSucceeded` 

### Ý chính cần nhớ

* Dùng `OperatorDegraded=False` để biểu thị trạng thái thành công.
* Điều này hợp lệ, nhưng dễ gây nhầm khi đọc nhanh.
* Tên condition và cách dùng True/False phải được thiết kế rất cẩn thận để dễ hiểu. 

---

## Using the OLM OperatorCondition

### Kiến thức cần nhớ

* Khi Operator được OLM quản lý, OLM tạo một CR tên là `OperatorCondition`.
* Operator có thể ghi condition vào đó để **ảnh hưởng đến cách OLM quản lý upgrade**. 

### YAML mẫu trong sách

```yaml
apiVersion: operators.coreos.com/v1

kind: OperatorCondition

metadata:

  name: sample-operator

  namespace: operator-ns

status:

  conditions:

  - type: Upgradeable

    status: False

    reason: "OperatorBusy"

    message: "Operator is currently busy with a critical task"

    lastTransitionTime: "2022-01-19T12:00:00Z"
```



### Giải thích chi tiết

* `type: Upgradeable`

  * condition chuẩn mà OLM hiểu
* `status: False`

  * báo rằng Operator hiện **không nên được upgrade**
* `reason` / `message`

  * diễn giải lý do chặn upgrade

### Code sách sử dụng trong `Reconcile()`

```go
import (

...

   apiv2 "github.com/operator-framework/api/pkg/operators/v2"

   "github.com/operator-framework/operator-lib/conditions"

)

func (r *NginxOperatorReconciler) Reconcile(ctx context.
Context, req ctrl.Request) (ctrl.Result, error) {

...

  condition, err := conditions.InClusterFactory{r.Client}.

   NewCondition(apiv2.ConditionType(apiv2.Upgradeable))

  if err != nil {

   return ctrl.Result{}, err

  }

  err = condition.Set(ctx, metav1.ConditionTrue,

   conditions.WithReason("OperatorUpgradeable"),

   conditions.WithMessage("The operator is upgradeable"))

  if err != nil {

   return ctrl.Result{}, err

  }

...

}
```



### Giải thích chi tiết code

* `apiv2 "github.com/operator-framework/api/pkg/operators/v2"`

  * import các type/hằng condition của Operator Framework
* `"github.com/operator-framework/operator-lib/conditions"`

  * thư viện helper để thao tác với `OperatorCondition`
* `conditions.InClusterFactory{r.Client}`

  * tạo factory dùng Kubernetes client hiện tại
* `NewCondition(apiv2.ConditionType(apiv2.Upgradeable))`

  * tạo đối tượng condition kiểu `Upgradeable`
* `condition.Set(...)`

  * ghi condition đó vào `OperatorCondition`
* `metav1.ConditionTrue`

  * báo rằng Operator hiện **có thể nâng cấp**
* `conditions.WithReason(...)`, `conditions.WithMessage(...)`

  * gắn metadata bổ sung cho condition

### YAML `overrides` trong sách

```yaml
apiVersion: operators.coreos.com/v1

kind: OperatorCondition

metadata:

  name: sample-operator

  namespace: operator-ns

spec:

  overrides:

  - type: Upgradeable

    status: True

    reason: "OperatorIsStable"

    message: "Forcing an upgrade to bypass bug state"

status:

  conditions:

  - type: Upgradeable

    status: False

    reason: "OperatorBusy"

    message: "Operator is currently busy with a critical task"

    lastTransitionTime: "2022-01-19T12:00:00Z"
```



### Giải thích chi tiết

* `spec.overrides`

  * cho phép admin override condition mà Operator đang báo
* Trường hợp dùng:

  * biết đang có bug hoặc trạng thái tạm thời
  * vẫn muốn ép cho phép upgrade

---

## Implementing metrics reporting

### Kiến thức cần nhớ

* Metrics là nền tảng của observability trong cluster.
* Trong Capability Model, metrics là thành phần rất quan trọng để tiến tới **Level IV – Deep Insights**.
* Nhiều controller native của Kubernetes đã tự export metrics cho chính chúng.

### Các khái niệm chính

* **Core metrics**

  * metric chung như CPU/memory usage
  * thường do `metrics-server` cung cấp
* **Service metrics**

  * metric riêng cho từng ứng dụng/component
  * do chính ứng dụng định nghĩa trong code
  * có thể scrape bằng Prometheus/OpenTelemetry rồi visualize bằng Grafana 

### Điều cần nhớ

* Boilerplate project đã có sẵn metrics endpoint.
* Bạn chỉ cần tập trung định nghĩa metric và register/update chúng. 

---

## Adding a custom service metric

### Kiến thức cần nhớ

* `controller-runtime` đã có sẵn nhiều built-in metrics với prefix `controller_runtime_`.

### Built-in metrics được sách nêu

Trong package `sigs.k8s.io/controller-runtime/pkg/internal/controller/metrics` có 7 metrics Prometheus chính cho controller-runtime. Chúng dùng label `controller`, và riêng `controller_runtime_reconcile_total` có thêm label `result`.

1. `controller_runtime_reconcile_total{controller, result}`

Đây là **counter** đếm tổng số lần reconcile đã chạy xong, chia theo kết quả:

* `result="success"`: reconcile xong, không lỗi, không yêu cầu requeue
* `result="error"`: reconcile trả về `error`
* `result="requeue"`: reconcile không lỗi nhưng trả `Requeue=true`
* `result="requeue_after"`: reconcile không lỗi nhưng trả `RequeueAfter > 0`

Cách đọc:

* Đây là metric “volume + outcome” tổng quát nhất.
* Tỷ lệ `error / total` cho biết controller có ổn định không.
* `requeue` hoặc `requeue_after` cao thường nghĩa là controller đang polling, retry nhiều, hoặc logic reconcile chưa hội tụ nhanh.

2. `controller_runtime_reconcile_errors_total{controller}`

Đây là **counter** đếm tổng số lần reconcile trả về lỗi. Mỗi khi `err != nil`, metric này tăng 1.

Cách đọc:

* Nếu metric này tăng đều, controller đang fail trên đường reconcile.
* Nó là “subset” của `reconcile_total{result="error"}` về mặt ý nghĩa thực tế; cả hai đều tăng khi có lỗi. `reconcile_errors_total` tiện cho alert đơn giản, còn `reconcile_total` tiện để breakdown theo outcome.

3. `controller_runtime_terminal_reconcile_errors_total{controller}`

Đây cũng là **counter**, nhưng chỉ đếm các lỗi kiểu **terminal error**. Trong code, nó tăng khi `errors.Is(err, reconcile.TerminalError(nil))` là true. Nghĩa là controller coi đây là lỗi “không nên retry bằng rate-limited queue nữa”.

Cách đọc:

* Nếu metric này tăng, lỗi không phải transient kiểu “thử lại rồi sẽ hết”, mà thường là lỗi logic, dữ liệu đầu vào, validation, hoặc trạng thái không thể xử lý tiếp.
* Nó vẫn đồng thời làm tăng `reconcile_errors_total` và `reconcile_total{result="error"}`.

4. `controller_runtime_reconcile_panics_total{controller}`

Đây là **counter** đếm số lần code trong reconciler bị `panic`. Nó tăng trong `defer` recovery block của `Controller.Reconcile`. Nếu `RecoverPanic` bật, panic sẽ được recover và biến thành error; nếu không thì panic bị ném lại. Nhưng metric panic vẫn tăng trước.

Cách đọc:

* Chỉ cần > 0 là đáng chú ý.
* Đây thường là bug thật sự trong code, không phải lỗi nghiệp vụ bình thường.
* Nên alert riêng cho metric này.

5. `controller_runtime_reconcile_time_seconds{controller}`

Đây là **histogram** đo thời gian mỗi lần reconcile, đơn vị là giây. Code ghi nhận bằng `Observe(reconcileTime.Seconds())`. Bucket trải từ 5ms đến 60s, và còn cấu hình native histogram.

Cách đọc:

* Dùng để xem p50, p95, p99 latency của reconcile.
* Nếu p95 tăng nhưng error không tăng, có thể controller chậm do API server, external dependency, cache sync, hoặc xử lý quá nhiều object con.
* Nếu latency cao kèm `active_workers` gần chạm `max_concurrent_reconciles`, controller có thể đang bị nghẽn throughput.

PromQL hay dùng:

```promql
histogram_quantile(
  0.95,
  sum by (le, controller) (
    rate(controller_runtime_reconcile_time_seconds_bucket[5m])
  )
)
```

6. `controller_runtime_max_concurrent_reconciles{controller}`

Đây là **gauge** biểu diễn cấu hình `MaxConcurrentReconciles` của controller. Nó được set khi init metrics, không phải số worker đang chạy thật tại thời điểm đó.

Cách đọc:

* Đây là “capacity cấu hình”.
* Dùng làm mẫu số để so với `active_workers`.
* Nếu giá trị thấp, controller xử lý tuần tự hoặc rất ít song song; nếu cao, controller có khả năng ăn queue nhanh hơn nhưng cũng tạo thêm áp lực lên API server / external systems.

7. `controller_runtime_active_workers{controller}`

Đây là **gauge** đếm số worker đang bận xử lý reconcile tại thời điểm hiện tại. Mỗi khi worker lấy item ra xử lý thì +1, xong thì -1.

Cách đọc:

* Nếu metric thường xuyên gần bằng `max_concurrent_reconciles`, controller đang chạy gần full công suất.
* Nếu queue backlog lớn và `active_workers ~= max_concurrent_reconciles`, bạn có thể cần tăng concurrency hoặc giảm thời gian mỗi reconcile.
* Nếu `active_workers` thường thấp nhưng reconcile vẫn chậm, bottleneck có thể nằm ở event rate thấp, queue policy, hoặc external wait.

8. `controller_runtime_reconcile_timeouts_total{controller}`

Đây là **counter** mới hơn, đếm số reconcile bị timeout do `ReconciliationTimeout` của wrapper context trong controller-runtime. Code kiểm tra rất chặt: chỉ tăng khi context thật sự `DeadlineExceeded` và nguyên nhân timeout đúng là wrapper timeout của controller-runtime, không phải parent context bị cancel hay timeout từ nơi khác. Release notes của controller-runtime cũng ghi metric này được thêm vào để track `ReconciliationTimeout`.

Cách đọc:

* Nếu metric này tăng, reconcile của bạn đang vượt quá timeout cấu hình.
* Nó hữu ích để phân biệt:

  * reconcile chậm nói chung (`reconcile_time_seconds` cao)
  * reconcile bị timeout thật sự (`reconcile_timeouts_total` tăng)
* Nếu timeout tăng cùng với latency p95/p99 tăng, gần như chắc là timeout threshold đang bị vượt.


### Code sách sử dụng: `controllers/metrics/metrics.go`

```go
package metrics

import (

   "github.com/prometheus/client_golang/prometheus"

   "sigs.k8s.io/controller-runtime/pkg/metrics"

)

var (

   ReconcilesTotal = prometheus.NewCounter(

      prometheus.CounterOpts{

         Name: "reconciles_total",

         Help: "Number of total reconciliation attempts",

      },

   )

)

func init() {

   metrics.Registry.MustRegister(ReconcilesTotal)

}
```



### Giải thích chi tiết code

* `package metrics`

  * tách metric sang package riêng để dễ quản lý
* `prometheus.NewCounter(...)`

  * tạo metric kiểu counter
  * counter chỉ tăng, không giảm
* `Name: "reconciles_total"`

  * tên metric sẽ được Prometheus scrape
* `Help: ...`

  * mô tả metric
* `metrics.Registry.MustRegister(...)`

  * đăng ký collector với global registry của controller-runtime
  * `MustRegister` sẽ panic nếu đăng ký lỗi/trùng

### Metrics naming best practices từ sách

* Nên có prefix đặc thù cho ứng dụng thật
* Counter tích lũy nên có hậu tố `_total` 

### Code sách thêm vào `Reconcile()`

```go
controllers/nginxoperator_controller.go:

import (

...

  "github.com/sample/nginx-operator/controllers/metrics"

...

)

func (r *NginxOperatorReconciler) Reconcile(...) (...) {

  metrics.ReconcilesTotal.Inc()

  ...

}
```



### Giải thích chi tiết code

* import package `metrics`
* gọi `metrics.ReconcilesTotal.Inc()` ở đầu `Reconcile()`
* mục tiêu:

  * tăng counter cho **mọi lần** reconcile attempt
  * không phân biệt thành công hay thất bại

### Ý chính cần nhớ

* Vị trí tăng metric rất quan trọng.
* Đặt ở đầu `Reconcile()` nghĩa là đang đo **attempt**, không phải **success**.
* Sách so sánh metric custom này với built-in `controller_runtime_reconcile_total` và cho thấy giá trị là giống nhau. 

---

## RED metrics

### Kiến thức cần nhớ

Operator SDK docs khuyến nghị phương pháp **RED**, gồm 3 loại metric cốt lõi mà mọi service nên expose: 

* **Rate**
* **Errors**
* **Duration**

### Ý nghĩa từng loại

* **Rate**

  * số request/attempt mỗi giây
  * giúp phát hiện requeue quá nhiều, hotloop, pipeline chưa tối ưu
* **Errors**

  * số lần thất bại
  * khi kết hợp với Rate sẽ cho biết tỷ lệ lỗi và pattern lỗi
* **Duration**

  * thời gian hoàn thành công việc
  * giúp phát hiện latency cao, hiệu năng kém, hoặc cluster đang suy giảm sức khỏe 

### Ý chính cần nhớ

* RED là điểm khởi đầu rất tốt nếu bạn chưa biết đo gì.
* Từ các metric RED có thể suy ra hoặc xây dựng:

  * SLI
  * SLO
  * alerts
  * dashboard vận hành cơ bản 

---

## Implementing leader election

### Kiến thức cần nhớ

* Khi có nhiều replica Operator cùng chạy, chỉ nên có **một leader** thực hiện reconciliation.
* Các replica còn lại đứng chờ để takeover khi leader mất khả dụng.
* Leader election giúp:

  * tránh nhiều instance cùng reconcile một workload
  * hỗ trợ rolling upgrades
  * hỗ trợ failover tốt hơn

### Code sách sử dụng trong `main.go`

```go
func main() {

...

   var enableLeaderElection bool

   flag.BoolVar(&enableLeaderElection, "leader-elect", false,

      "Enable leader election for controller manager. "+

         "Enabling this will ensure there is only one active 
controller manager.")

...

   mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.
Options{

...

      HealthProbeBindAddress: probeAddr,

      LeaderElection:         enableLeaderElection,

      LeaderElectionID:       "df4c7b26.example.com",

   })

   if err != nil {

      setupLog.Error(err, "unable to start manager")

      os.Exit(1)

   }
```



### Giải thích chi tiết code

* `flag.BoolVar(&enableLeaderElection, "leader-elect", false, ...)`

  * tạo CLI flag `--leader-elect`
  * mặc định `false`
* `LeaderElection: enableLeaderElection`

  * bật/tắt leader election theo flag
* `LeaderElectionID: "df4c7b26.example.com"`

  * định danh resource lock
  * các replica cùng `LeaderElectionID` sẽ tranh leadership trên cùng lock

### Runtime flag trong chương

```bash
--leader-elect
```

### Giải thích chi tiết

* Nếu binary chạy với `--leader-elect`

  * chỉ một controller manager active
* Nếu không bật

  * nhiều instance có thể cùng xử lý reconcile
  * dễ sinh contention hoặc race nếu Operator không thiết kế để multi-active

### Hai chiến lược sách nêu

* **Leader-with-lease**

  * leader phải định kỳ renew lease
  * failover nhanh
  * có nguy cơ split-brain
* **Leader-for-life**

  * leader chỉ mất quyền khi bị xóa và lock bị GC
  * tránh tranh chấp leader
  * failover có thể chậm hơn đáng kể 

### Ý chính cần nhớ

* Leader election là một trong các nền tảng quan trọng để tăng độ sẵn sàng của Operator.
* Đặc biệt hữu ích khi Operator có nhiều replica để phục vụ HA.

---

## Adding health checks

### Kiến thức cần nhớ

* Health check và readiness check là endpoint để kubelet / load balancer / hệ thống vận hành xác định:

  * tiến trình còn sống không
  * tiến trình đã sẵn sàng phục vụ hay chưa
* Operator SDK scaffold sẵn các check cơ bản này trong `main.go`.

### Code sách sử dụng

```go
main.go:

    if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err 
!= nil {

      setupLog.Error(err, "unable to set up health check")

      os.Exit(1)

   }

   if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != 
nil {

      setupLog.Error(err, "unable to set up ready check")

      os.Exit(1)

   }

   setupLog.Info("starting manager")

   if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {

      setupLog.Error(err, "problem running manager")

      os.Exit(1)

   }

}
```



### Giải thích chi tiết code

* `mgr.AddHealthzCheck("healthz", healthz.Ping)`

  * đăng ký check liveness/health
* `mgr.AddReadyzCheck("readyz", healthz.Ping)`

  * đăng ký check readiness
* `healthz.Ping`

  * no-op check luôn trả thành công
* `mgr.Start(ctrl.SetupSignalHandler())`

  * start manager sau khi đã đăng ký probes

### Signature của check function trong sách

```go
func(*http.Request) error
```



### Giải thích chi tiết

* nhận `*http.Request`
* trả `error`
* trả `nil` → check pass
* trả `error` → check fail

### Ý chính cần nhớ

* `healthz.Ping` chỉ là baseline rất tối thiểu.
* Trong production, bạn thường cần custom checks có ý nghĩa hơn, ví dụ:

  * dependency đã accessible chưa
  * cache đã sync chưa
  * client tới API server còn hoạt động không
* Sách nhấn mạnh bạn có thể thêm nhiều check bằng cách gọi lại `AddHealthzCheck` hoặc `AddReadyzCheck`; các check sẽ chạy tuần tự và nếu có check nào fail thì endpoint sẽ trả HTTP 500.

---

## Mapping nhanh với Capability Model

* **Metrics** → rất quan trọng để tiến lên **Level IV – Deep Insights**
* **Leader election / failover mindset** → hỗ trợ hướng tới **Level III – Full Lifecycle**
* **Status conditions / health checks** → cải thiện observability và debugging, làm Operator dễ vận hành hơn trong thực tế 