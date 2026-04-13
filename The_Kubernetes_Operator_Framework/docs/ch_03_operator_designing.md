# Designing an Operator: CRD, API, and Target Reconciliation

## 1. Mục tiêu

Hướng dẫn cách **thiết kế một Operator trước khi bắt tay viết code**. Ví dụ xuyên suốt là một Operator quản lý một ứng dụng `nginx` đơn giản. Các bước thiết kế mà đi qua là:

1. mô tả bài toán,
2. thiết kế API và CRD,
3. chuẩn bị các resource phụ trợ,
4. thiết kế reconciliation loop,
5. nghĩ trước chuyện upgrade/downgrade,
6. thiết kế cách báo lỗi. 

---

## 2. Bài toán mẫu

### User story

```text
As a cluster administrator, I want to use an Operator to manage my nginx
application so that its health and monitoring are automatically managed for me.
```

### Diễn giải

* **User**: cluster administrator
* **Action**: dùng Operator để quản lý ứng dụng nginx
* **Reason**: muốn health và monitoring của ứng dụng được tự động quản lý. 

### MVP (Minimum Viable Product) của Operator ví dụ

Operator cơ bản này cần làm được 3 việc:

* triển khai ứng dụng,
* giữ ứng dụng tiếp tục chạy nếu bị lỗi,
* báo cáo tình trạng sức khỏe của ứng dụng. 

### Capability level mục tiêu

Theo Capability Model:

* nếu Operator cài được Operand, quản lý tài nguyên cần thiết, có CRD để cấu hình và có thể báo trạng thái thì đạt **Level I**,
* nếu xử lý được nâng cấp nữa thì có thể lên **Level II**. 

---

## 3. API và CRD: giao diện của Operator

**Custom Resource (CR)** là giao diện mà người dùng sẽ tương tác để điều khiển Operator.
**CustomResourceDefinition (CRD)** là bản thiết kế cho loại object đó. Nói cách khác:

* **CRD** = kiểu tài nguyên tùy biến
* **CR** = instance cụ thể của kiểu đó. 

Tư tưởng chính của phần này là:

* người dùng **không nên phải sửa trực tiếp Deployment/Pod** của Operand,
* thay vào đó họ làm việc với **CR** của Operator,
* Operator sẽ chuyển ý định của người dùng thành thay đổi trên các resource Kubernetes thật. 

---

## 4. Các quy ước thiết kế API Kubernetes cần nhớ

CRD tuy là “custom” nhưng vẫn sống trong hệ sinh thái API của Kubernetes, vì vậy nên tuân theo các quy ước quen thuộc. 

### 4.1 `kind`

* Xác định **loại object**.
* Ví dụ: `MyOperator`, `NginxOperator`.
* Client và API server dùng nó để biết object này là loại gì. 

### 4.2 `apiVersion`

* Xác định **phiên bản API** của object.
* Ví dụ trong sách: `v1alpha1`, `v1beta1`, `v1`.
* Với custom resource thực tế, thường là dạng `group/version`, ví dụ `operator.example.com/v1alpha1`. 

### 4.3 `resourceVersion`

* Là trường nội bộ do API server dùng để theo dõi mỗi lần object bị sửa.
* Quan trọng cho **concurrency control**.
* Ví dụ sách nêu: khi gọi `Get()` rồi `Update()`, API server có thể từ chối update nếu `resourceVersion` trên server đã đổi do controller khác vừa sửa object trước đó. 

### 4.4 `generation`

* Khác với `resourceVersion`.
* Dùng để theo dõi các thay đổi **có ý nghĩa về mặt spec**.
* Ví dụ: một Deployment rollout phiên bản mới thì `generation` thay đổi. 

### 4.5 `creationTimestamp`

* Cho biết thời điểm object được tạo.
* Có thể dùng để suy ra “tuổi” của object. 

### 4.6 `deletionTimestamp`

* Cho biết object đã nhận yêu cầu xóa từ API server hay chưa. 

### 4.7 `labels`

* Dùng để **phân loại, nhóm, lọc** object.
* Phù hợp với các truy vấn chọn lọc qua API. 

### 4.8 `annotations`

* Cũng là metadata, nhưng thiên về **mô tả bổ sung**, không phải để lọc nhóm như labels. 

### 4.9 `spec`

* Chứa **trạng thái mong muốn** mà người dùng khai báo.
* Với Operator, đây là nơi người dùng đưa cấu hình cho Operand. 

### 4.10 `status`

* Chứa **trạng thái quan sát được hiện tại**.
* Do Operator/controller cập nhật. 

### 4.11 `status.conditions`

Các nguyên tắc cho conditions:

* condition phải **tự mô tả được**,
* không nên đổi nghĩa sau khi đã định nghĩa,
* có thể dùng `True` hoặc `False` làm “trạng thái bình thường”, miễn dễ đọc,
* nên phản ánh **current known state**, không chỉ ghi lại việc “vừa chuyển trạng thái”. 

Ví dụ trong sách:

* `Ready=true` dễ đọc hơn `NotReady=false`. 

### 4.12 Sub-object nên là list, không phải map

Nếu cần nhiều port, nên dùng dạng list có cấu trúc thay vì map.
Ví dụ tư tưởng là:

```yaml
ports:
  - name: http
    port: 80
  - name: https
    port: 443
```

thay vì map kiểu `http: 80`, `https: 443`. 

### 4.13 Field optional nên dùng pointer

Trong Go, field optional nên dùng pointer để phân biệt:

* giá trị zero hợp lệ,
* với trường chưa được set. 

### 4.14 Tên field có đơn vị nên ghi rõ đơn vị

Ví dụ:

```text
restartTimeoutSeconds
```

thay vì một tên mơ hồ. 

---

# 5. Cấu trúc của một CRD

Ví dụ CRD:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: myoperator.operator.example.com
spec:
  group: operator.example.com
  names:
    kind: MyOperator
    listKind: MyOperatorList
    plural: myoperators
    singular: myoperator
  scope: Namespaced
  versions:
    - name: v1alpha1
      schema:
        openAPIV3Schema:
         ...
      served: true
      storage: true
      subresources:
        status: {}
status:
  acceptedNames:
    kind: ""
    plural: ""
  conditions: []
  storedVersions: []
```



## 5.1 Giải thích từng trường

### `metadata.name`

* **Tên của CRD**, không phải tên của CR.
* Ví dụ truy xuất CRD này:

```bash
kubectl get crd/myoperator.operator.example.com
```



### `spec.group`

* API group cho custom resource.
* Mục tiêu là tránh xung đột với các API khác trong cluster. 

### `spec.names.kind`

* Tên loại object ở dạng singular/camel case.
* Ví dụ: `MyOperator`. 

### `spec.names.listKind`

* Tên loại list của object.
* Ví dụ: `MyOperatorList`. 

### `spec.names.plural`

* Dạng số nhiều để thao tác qua kubectl/API.
* Ví dụ: `myoperators`. 

### `spec.names.singular`

* Dạng số ít.
* Ví dụ: `myoperator`. 

### `spec.scope`

* Phạm vi của resource:

  * `Namespaced`
  * hoặc `Cluster`. 

### `spec.versions`

* Danh sách các version mà CRD phục vụ.
* Cho phép hỗ trợ nhiều version song song trong giai đoạn chuyển tiếp.
* Chỉ một version được đặt là storage version. 

### `schema.openAPIV3Schema`

* Schema validation của CR.
* Dùng OpenAPI v3 structural schema.
* Không nên viết tay nếu schema phức tạp; nên dùng tool sinh như Kubebuilder/controller-gen. 

### `served`

* Version này có được serve qua REST API không. 

### `storage`

* Version này có phải là storage version không.
* Chỉ có một version được đặt `storage: true`. 

### `subresources.status`

* Bật `status` subresource để controller/Operator có thể cập nhật trạng thái tách biệt khỏi `spec`. 

---

# 6. CRD mẫu cho `NginxOperator`

Chọn 3 field cấu hình chính trong `spec` của Operator này:

## 6.1 `port`

* Cổng mà Pod nginx sẽ expose trong cluster.
* Giúp đổi cổng qua CR thay vì sửa thẳng Pod/Deployment. 

## 6.2 `replicas`

* Số replica cho Deployment. 

## 6.3 `forceRedeploy`

* Một field gần kiểu no-op.
* Đổi giá trị field này sẽ kích hoạt Operator rollout lại Operand mà không cần thay đổi cấu hình nghiệp vụ thật.
* Hữu ích khi Deployment bị “kẹt” và cần ép redeploy. 

## 6.4 Ví dụ CR

```yaml
apiVersion: v1alpha1
kind: NginxOperator
metadata:
  name: instance
spec:
  port: 80
  replicas: 1
status:
  ...
```

> Ghi chú thực tế: khi tạo CR thật, thường dùng `apiVersion` đầy đủ kiểu `operator.example.com/v1alpha1`. 

## 6.5 Lệnh thao tác

```bash
kubectl get -o yaml nginxoperator/instance
```

Mục đích: lấy object CR dưới dạng YAML nếu CRD được định nghĩa đúng. 

---

# 7. Các resource phụ trợ mà Operator còn phải quản lý

Ngoài CRD, Operator ví dụ còn phải biết về:

* Deployment của nginx Operand,
* ServiceAccount,
* Role,
* RoleBinding. 

Câu hỏi: **Operator lấy định nghĩa các resource đó ở đâu?**

---

## 7.1 Cách 1: dựng resource trực tiếp bằng Go struct

Ví dụ:

```go
import appsv1 "k8s.io/api/apps/v1"

nginxDeployment := &appsv1.Deployment{
  TypeMeta: metav1.TypeMeta{
    Kind: "Deployment",
    apiVersion: "apps/v1",
  },
  ObjectMeta: metav1.ObjectMeta{
    Name: "nginx-deploy",
    Namespace: "nginx-ns",
  },
  Spec: appsv1.DeploymentSpec{
    Replicas: 1
    Selector: &metav1.LabelSelector{
      MatchLabels: map[string]string{"app":"nginx"},
    },
    Template: v1.PodTemplateSpec{
      Spec: v1.PodSpec{
        ObjectMeta: metav1.ObjectMeta{
          Name: "nginx-pod",
          Namespace: "nginx-ns",
          Labels: map[string]string{"app":"nginx"},
        },
        Containers: []v1.Container{
          {
             Name: "nginx",
             Image: "nginx:latest",
             Ports: []v1.ContainerPort{{ContainerPort: int32(80)}},
          },
        },
      },
    },
  },
}
```



### Ý nghĩa các trường trong ví dụ trên

* `TypeMeta.Kind`: loại resource, ở đây là `Deployment`
* `TypeMeta.apiVersion`: API version của Deployment
* `ObjectMeta.Name`: tên Deployment
* `ObjectMeta.Namespace`: namespace chứa Deployment
* `Spec.Replicas`: số replica
* `Selector.MatchLabels`: selector để Deployment biết Pod nào thuộc nó
* `Template`: Pod template mà Deployment sẽ tạo
* `Template.ObjectMeta.Labels`: labels gắn lên Pod
* `Containers[].Name`: tên container
* `Containers[].Image`: image chạy
* `Containers[].Ports`: cổng container. 

### Ưu điểm

* sẵn sàng để truyền vào Kubernetes API client,
* rất tiện cho code.

### Nhược điểm

* khó đọc,
* không thân thiện bằng YAML đối với người quen làm việc với Kubernetes manifest. 

---

## 7.2 Cách 2: giữ manifest ở dạng YAML rồi nhúng vào binary

Ví dụ YAML đơn giản hơn:

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx-deploy
  namespace: nginx-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
         image: nginx:latest
         ports:
          - containerPort: 80
```



### Ý nghĩa

* Cùng một resource như ví dụ Go struct ở trên,
* nhưng dễ đọc và dễ bảo trì hơn,
* tiện cho kiểm tra CI và tổ chức nhiều version resource khác nhau trong mã nguồn. 

### Hai cách nhúng mà sách nhắc tới

* `go-bindata`
* `go:embed` (Go 1.16+). 

### Ghi chú thực tế hiện nay

Nếu làm project Go mới, nên ưu tiên `go:embed`. `go-bindata` chủ yếu là hướng cũ hơn.

---

# 8. Thiết kế reconciliation loop

Đây là phần cốt lõi của Operator.

## 8.1 Reconciliation loop là gì?

Operator hoạt động bằng cách **so khớp current state của cluster với desired state do người dùng khai báo**.
Các lần kiểm tra này thường được kích hoạt bởi event liên quan đến Operand, ví dụ tạo hoặc xóa Pod trong namespace mục tiêu. 

---

## 8.2 Level-based vs Edge-based

### Edge-based

* chỉ xử lý dựa trên chính event vừa xảy ra,
* hiệu quả hơn,
* nhưng dễ sai nếu mất event. 

### Level-based

* mỗi lần reconcile sẽ đánh giá lại trạng thái hiện tại của hệ thống,
* đáng tin cậy hơn,
* phù hợp hơn với hệ phân tán lớn như Kubernetes. 

### Kết luận của sách

Operator nên đi theo **level-based triggering**. 

---

## 8.3 Chữ ký hàm `Reconcile()`

Chữ ký điển hình:

```go
func (r *Controller) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
```



### Ý nghĩa các thành phần

* `r *Controller`: receiver của controller
* `ctx context.Context`: context cho lifecycle của request
* `req ctrl.Request`: định danh object cần reconcile
* `ctrl.Result`: cho controller biết có cần requeue hay không
* `error`: báo lỗi nếu reconcile thất bại. 

---

## 8.4 Pseudocode reconcile loop trong sách

```text
func Reconcile:

  // Get the Operator's CRD, if it doesn't exist then return
  // an error so the user knows to create it

  operatorCrd, error = getMyCRD()

  if error != nil {
    return error
  }

  // Get the related resources for the Operator (ie, the
  // Operand's Deployment). If they don't exist, create them

  resources, error = getRelatedResources()

  if error == ResourcesNotFound {
    createRelatedResources()
  }

  // Check that the related resources relevant values match
  // what is set in the Operator's CRD. If they don't match,
  // update the resource with the specified values.

  if resources.Spec != operatorCrd.Spec {
    updateRelatedResources(operatorCrd.Spec)
  }
```

### Ý nghĩa từng bước

#### Bước 1: lấy CR của Operator

* Nếu CR chưa có thì không nên tự tạo.
* Best practice mà sách nhắc: Operator **không nên tự quản vòng đời CR của chính nó**. 

#### Bước 2: lấy các resource liên quan

* Ở ví dụ này là Deployment của Operand.
* Nếu chưa có thì tạo. 

#### Bước 3: so sánh current state với desired state

* Nếu khác nhau thì update. 

### Vì sao không update mỗi lần?

Tránh update vô điều kiện vì:

* tạo quá nhiều API call không cần thiết,
* tăng nguy cơ **update hot loop**: chính update của Operator lại tạo event mới và kích hoạt reconcile tiếp. 

---

# 9. Handling upgrades and downgrades

Khi nói về versioning, cần nghĩ đến hai thứ:

* version của Operand,
* version của Operator.

## 9.1 Upgrade Operand

Với ví dụ nginx, việc nâng cấp Operand khá trực tiếp:

* kéo image tag mới,
* cập nhật Deployment của Operand. 

## 9.2 Upgrade Operator

Nếu code hoặc API của Operator thay đổi, cần có chiến lược versioning cho Operator.
OLM và CSV sẽ giúp phần này. 

## 9.3 Khi CRD thay đổi không tương thích ngược

Nếu CRD thay đổi theo cách phá compatibility, ví dụ bỏ field cũ:

* phải tăng API version, ví dụ `v1alpha1` → `v1alpha2` hoặc `v1beta1`
* ship version mới cùng version cũ một thời gian để người dùng chuyển dần. 

## 9.4 Storage version

* Trong nhiều version chỉ có **một storage version**.
* Khi loại bỏ version cũ, storage version có thể phải đổi theo. 

## 9.5 Ví dụ `StorageVersionMigration`

```yaml
apiVersion: migration.k8s.io/v1alpha1
kind: StorageVersionMigration
metadata:
  name: nginx-operator-storage-version-migration
spec:
  resource:
    group: operator.example.com
    resource: nginxoperators
    version: v1alpha2
```



### Ý nghĩa các trường

* `apiVersion`: API của migration object
* `kind`: loại object migration
* `metadata.name`: tên migration
* `spec.resource.group`: API group của CR cần migrate
* `spec.resource.resource`: plural name của resource
* `spec.resource.version`: target storage version cần migrate tới. 

### Ghi chú thực tế

Khái niệm storage version migration vẫn còn đúng, nhưng cách vận hành thực tế hiện nay cần xem theo version Kubernetes bạn đang dùng.

---

# 10. Failure reporting

Phải nghĩ đến lỗi ở cả:

* **Operand**
* và **Operator**. 

Khi lỗi xảy ra trong reconcile loop:

* framework có thể retry,
* có thể backoff dần,
* nhưng lỗi vẫn phải được **phơi bày ra cho người dùng**. 

Ba kênh đề xuất:

1. logging
2. status updates
3. events.

---

## 10.1 Logging

### Lệnh ví dụ

```bash
kubectl logs pod/my-pod
```

### Ý nghĩa

* xem log của Pod Operator hay Operand. 

### Ưu điểm

* đơn giản,
* tiện để debug flow xử lý.

### Nhược điểm

* log Pod không bền,
* có thể bị mất khi Pod bị dọn,
* log thường rất ồn và thiếu ngữ cảnh cho người vận hành. 

---

## 10.2 Status updates / Conditions

Ví dụ condition trong `status`:

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
```

### Ý nghĩa các trường

* `type`: tên condition, ví dụ `Ready`
* `status`: giá trị `True`, `False`, hoặc `Unknown`
* `lastProbeTime`: thời điểm probe gần nhất nếu dùng
* `lastTransitionTime`: lúc condition đổi trạng thái lần gần nhất.

### Ý tưởng áp dụng trong ví dụ nginx Operator

* đặt `Ready=True` khi Operator chạy ổn và reconcile thành công,
* đổi sang `Ready=False` khi có lỗi,
* kèm `Reason` để giải thích. 

---

## 10.3 Events

### Lệnh ví dụ

```bash
kubectl describe pod/coredns-558bd4d5db-6mqc2 -n kube-system
kubectl get events
```

### Ý nghĩa

Events là object native của Kubernetes, có thể dùng để báo:

* lỗi,
* warning,
* hoặc thành công ở các mốc quan trọng. 

### Các field cần quan tâm

* `Type`: `Normal` hoặc `Warning`
* `Reason`: lý do
* `Message`: mô tả dễ đọc
* `Count`: số lần xảy ra
* `ReportingController`: controller phát event. 

### Ví dụ events

* `FailedScheduling`
* `Scheduled`
* `Pulled`
* `Created`
* `Started`