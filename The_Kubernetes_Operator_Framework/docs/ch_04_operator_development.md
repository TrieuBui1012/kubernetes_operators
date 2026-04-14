# Developing an Operator with the Operator SDK

## Setting up your project

### 1. Tạo thư mục project

```bash
mkdir nginx-operator
cd nginx-operator
```

### Giải thích

- `mkdir nginx-operator`: tạo thư mục project rỗng.
- `cd nginx-operator`: chuyển vào thư mục đó để scaffold sinh file đúng vị trí.

Đây là bước rất cơ bản, nhưng tư duy quan trọng ở đây là: **Operator SDK không thêm tính năng vào repo đang có sẵn một cách mơ hồ**, mà thường scaffold từ một thư mục project rõ ràng.

---

### 2. Khởi tạo boilerplate project

```bash
operator-sdk init --domain example.com --repo github.com/example/nginx-operator
```

### Giải thích chi tiết command

- `operator-sdk init`: khởi tạo boilerplate project cho Operator.
- `--domain example.com`: domain để ghép thành API group về sau, ví dụ `operator.example.com`.
- `--repo github.com/example/nginx-operator`: khai báo module path/repository path cho project Go.

### Kết quả sau khi init

Chạy `ls` để xem các file được tạo:

```bash
ls
```

Các thành phần quan trọng mà chapter nhấn mạnh:

- `config/`: nơi chứa YAML/manifests và các file kustomize liên quan.
- `hack/`: script phụ trợ cho generate/verify.
- `.dockerignore`, `.gitignore`
- `Dockerfile`
- `Makefile`
- `PROJECT`
- `go.mod`, `go.sum`
- `cmd/main.go`
- `api/`: định nghĩa các API
- `internal/`: chứa code xử lý của controller

### Trong đó

- `main.go` là entrypoint của Operator.
- `Makefile` là nơi gom workflow generate/build/test/deploy.
- `PROJECT` là metadata file của Kubebuilder.
- Sau bước này, project **compile được**, nhưng **chưa có API/CRD/controller logic thật sự**.

---

## Defining an API

### 1. Tạo API và controller skeleton

```bash
operator-sdk create api --group operator --version v1alpha1 --kind NginxOperator --resource --controller
```

### Giải thích chi tiết command

- `create api`: scaffold API types.
- `--group operator`: API group.
- `--version v1alpha1`: version đầu tiên của CRD/API.
- `--kind NginxOperator`: Kind của custom resource.
- `--resource`: sinh resource scaffolding.
- `--controller`: sinh controller skeleton cho resource này.

### Kết quả chapter muốn bạn hiểu

Lệnh trên sinh ra:

- `api/v1alpha1/nginxoperator_types.go`
- `controllers/nginxoperator_controller.go`
- cập nhật `main.go` để manager đăng ký controller mới

### 2. Ba kiểu dữ liệu quan trọng

Sách chỉ ra ba type cốt lõi:

```go
type NginxOperatorSpec struct {
    Foo string `json:"foo,omitempty"`
}

type NginxOperatorStatus struct {
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
type NginxOperator struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   NginxOperatorSpec   `json:"spec,omitempty"`
    Status NginxOperatorStatus `json:"status,omitempty"`
}
```

### Giải thích

- `Spec`: biểu diễn **desired state**.
- `Status`: biểu diễn **observed state**.
- `NginxOperator`: object chính của CR.
- `+kubebuilder:subresource:status`: cho phép cập nhật `/status` riêng.

### 3. Các field thật của Spec trong chapter

```go
type NginxOperatorSpec struct {
    // Port is the port number to expose on the Nginx Pod
    Port *int32 `json:"port,omitempty"`

    // Replicas is the number of deployment replicas to scale
    Replicas *int32 `json:"replicas,omitempty"`

    // ForceRedploy is any string, modifying this field
    // instructs the Operator to redeploy the Operand
    ForceRedeploy string `json:"forceRedeploy,omitempty"`
}
```

### Giải thích từng field

- `Port *int32`: cho phép cấu hình port của Nginx container.
- `Replicas *int32`: cho phép scale deployment qua CR.
- `ForceRedeploy`: field no-op để tạo event và ép Operator rollout lại operand.

### Vì sao `Port` và `Replicas` lại là pointer?

Vì Operator cần phân biệt:

- user không set gì cả
- user thật sự set giá trị `0`

Pointer giúp phân biệt `nil` và zero value.

### 4. Ví dụ field required/default bằng Kubebuilder markers

```go
// Port is the port number to expose on the Nginx Pod
// +kubebuilder:default=8080
// +kubebuilder:validation:Required
Port int `json:"port"`
```

### Ý nghĩa

- `default=8080`: cấp default.
- `validation:Required`: bắt buộc field phải tồn tại.
- bỏ `omitempty`: field không còn optional.

### Điều cần ghi nhớ

- Operator API nên được mô hình hóa như một **Kubernetes API object chuẩn**, không phải chỉ là một struct cấu hình tùy hứng.
- Bạn phải quen với việc: **Go type là nguồn chân lý, CRD YAML là output generate**.

### Regenerate code sau khi sửa API

```bash
make generate
```

### Vì sao cần lệnh này?

- cập nhật các file generated như `zz_generated.deepcopy.go`
- nếu quên chạy, generated code có thể lệch với type thật

---

## Adding resource manifests

### 1. Generate CRD/manifests

```bash
make manifests
```

### Giải thích

- Sinh CRD YAML từ Go types.
- Sinh thêm một số manifest khác như RBAC dựa trên markers.

Ví dụ output nổi bật mà chapter đề cập:

- `config/crd/bases/operator.example.com_nginxoperators.yaml`
- `config/rbac/role.yaml`

### 2. CRD được generate ra trông như thế nào?

Sách cho một CRD YAML khá dài. Phần cần nhớ nhất là:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: nginxoperators.operator.example.com
spec:
  group: operator.example.com
  names:
    kind: NginxOperator
    listKind: NginxOperatorList
    plural: nginxoperators
    singular: nginxoperator
  scope: Namespaced
  versions:
    - name: v1alpha1
      schema:
        openAPIV3Schema:
          properties:
            spec:
              properties:
                forceRedploy:
                  type: string
                port:
                  type: integer
                replicas:
                  type: integer
      served: true
      storage: true
      subresources:
        status: {}
```

### Giải thích

- `group`: API group của resource.
- `names`: định nghĩa kind/plural/singular.
- `scope: Namespaced`: CR nằm theo namespace.
- `versions`: khai báo API version được serve.
- `openAPIV3Schema`: schema generate từ Go types + markers.
- `subresources.status`: bật status subresource.

### 3. RBAC role generate ra cho CRD

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manager-role
rules:
- apiGroups:
  - operator.example.com
  resources:
  - nginxoperators
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - operator.example.com
  resources:
  - nginxoperators/finalizers
  verbs:
  - update
- apiGroups:
  - operator.example.com
  resources:
  - nginxoperators/status
  verbs:
  - get
  - patch
  - update
```

### Điều cần ghi nhớ

- Controller phải có quyền đọc CR của chính nó và cập nhật `/status` nếu dùng status reporting.
- `make manifests` vẫn là lệnh quan trọng, nhưng nên hiểu nó là wrapper trên `controller-gen`.

---

## Additional manifests and BinData

### 1. Cách 1 – định nghĩa Deployment trực tiếp bằng Go

```go
nginxDeployment := &appsv1.Deployment{
    TypeMeta: metav1.TypeMeta{
        Kind:       "Deployment",
        APIVersion: "apps/v1",
    },
    ObjectMeta: metav1.ObjectMeta{
        Name:      "nginx-deploy",
        Namespace: "nginx-ns",
    },
    Spec: appsv1.DeploymentSpec{
        // ...
    },
}
```

### Giải thích

Ưu điểm:

- type-safe
- IDE hỗ trợ tốt

Nhược điểm:

- verbose
- khó đọc với người quen YAML Kubernetes hơn là Go structs

### 2. Cách 2 – giữ manifest dưới dạng YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: "nginx-deployment"
  namespace: "nginx-operator-ns"
  labels:
    app: "nginx"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: "nginx"
  template:
    metadata:
      labels:
        app: "nginx"
    spec:
      containers:
      - name: "nginx"
        image: "nginx:latest"
        command: ["nginx"]
```

### Ý nghĩa chapter muốn bạn hiểu

Giữ manifest ở YAML giúp:

- gần với cách Kubernetes user quen đọc
- dễ maintain
- dễ kiểm tra/validate bằng tooling khác

### 3. `go-bindata` (cách cũ)

```bash
go get -u github.com/go-bindata/go-bindata/...
```

```bash
go-bindata -o assets/assets.go assets/...
```

```bash
go-bindata -o assets/assets.go -ignore=\assets\/assets\.go assets/...
```

### Giải thích

- `go-bindata` sẽ generate mã Go từ file tĩnh.
- `Asset(...)` được dùng để lấy bytes của file đã embed.

Ví dụ trong sách:

```go
import "github.com/sample/nginx-operator/assets"

asset, err := assets.Asset("nginx_deployment.yaml")
if err != nil {
    // process object not found error
}
```

### 4. `go:embed` (cách mới hơn)

```go
import (
    "embed"
)

//go:embed assets/nginx_deployment.yaml
var deployment embed.FS
```

### Decode YAML thành Deployment object

```go
import (
    appsv1 "k8s.io/api/apps/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"
)

var (
    appsScheme = runtime.NewScheme()
    appsCodecs = serializer.NewCodecFactory(appsScheme)
)

func main() {
    if err := appsv1.AddToScheme(appsScheme); err != nil {
        panic(err)
    }

    deploymentBytes, err := deployment.ReadFile("assets/nginx_deployment.yaml")
    if err != nil {
        panic(err)
    }

    deploymentObject, err := runtime.Decode(
        appsCodecs.UniversalDecoder(appsv1.SchemeGroupVersion),
        deploymentBytes,
    )
    if err != nil {
        panic(err)
    }

    dep := deploymentObject.(*appsv1.Deployment)
    _ = dep
}
```

### Giải thích

- `AddToScheme`: đăng ký kiểu `Deployment` vào scheme.
- `ReadFile`: lấy bytes của file embedded.
- `runtime.Decode(...)`: parse bytes thành object Kubernetes.
- cast về `*appsv1.Deployment` để dùng typed object.

### 5. Tách helper vào package `assets`

```go
package assets

import (
    "embed"
    appsv1 "k8s.io/api/apps/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"
)

var (
    //go:embed manifests/*
    manifests embed.FS
    appsScheme = runtime.NewScheme()
    appsCodecs = serializer.NewCodecFactory(appsScheme)
)

func init() {
    if err := appsv1.AddToScheme(appsScheme); err != nil {
        panic(err)
    }
}

func GetDeploymentFromFile(name string) *appsv1.Deployment {
    deploymentBytes, err := manifests.ReadFile(name)
    if err != nil {
        panic(err)
    }
    deploymentObject, err := runtime.Decode(
        appsCodecs.UniversalDecoder(appsv1.SchemeGroupVersion),
        deploymentBytes,
    )
    if err != nil {
        panic(err)
    }
    return deploymentObject.(*appsv1.Deployment)
}
```

### Điều cần ghi nhớ

- Chapter này muốn bạn hiểu trade-off giữa **resource-in-code** và **resource-as-YAML**.
- Với sample Operator nhỏ, sách nghiêng về YAML + embed helper.

### Cập nhật 2026

- Với Go hiện đại, **ưu tiên `go:embed`** thay cho `go-bindata`.
- `panic()` trong ví dụ là để đơn giản hóa flow dạy học; trong code thật cần error handling rõ hơn.

---

## Writing a control loop

### 1. Reconcile skeleton

```go
func (r *NginxOperatorReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // your logic here

    return ctrl.Result{}, nil
}
```

### Ý nghĩa

- `Reconcile()` là hàm cốt lõi của controller.
- `req` chứa tên/namespace của object kích hoạt reconcile.
- `ctrl.Result{}, nil` nghĩa là reconcile thành công, không yêu cầu retry ngay.

### 2. Lấy CR object của Operator

```go
func (r *NginxOperatorReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    operatorCR := &operatorv1alpha1.NginxOperator{}
    err := r.Get(ctx, req.NamespacedName, operatorCR)
    if err != nil && errors.IsNotFound(err) {
        logger.Info("Operator resource object not found.")
        return ctrl.Result{}, nil
    } else if err != nil {
        logger.Error(err, "Error getting operator resource object")
        return ctrl.Result{}, err
    }
    return ctrl.Result{}, nil
}
```

### Giải thích

- `r.Get(...)`: đọc object CR hiện tại.
- `errors.IsNotFound(err)`: CR không tồn tại thì không coi là lỗi cần retry.
- lỗi khác: trả về `err` để controller-runtime requeue.

### 3. Lấy Deployment hiện tại hoặc chuẩn bị create mới

```go
deployment := &appsv1.Deployment{}
create := false
err = r.Get(ctx, req.NamespacedName, deployment)
if err != nil && errors.IsNotFound(err) {
    create = true
    deployment = assets.GetDeploymentFromFile("manifests/nginx_deployment.yaml")
} else if err != nil {
    logger.Error(err, "Error getting existing Nginx deployment.")
    return ctrl.Result{}, err
}
```

### Giải thích

- Nếu chưa có Deployment, load từ manifest mặc định.
- Nếu đã có, dùng object hiện tại để update.
- `create` quyết định cuối hàm sẽ gọi `Create()` hay `Update()`.

### Lưu ý

Ví dụ này đang dùng simplification: lookup operand Deployment bằng `req.NamespacedName`, tức ngầm giả định operand có cùng tên/namespace với CR.

### 4. Áp config từ CR xuống Deployment

```go
if operatorCR.Spec.Replicas != nil {
    deployment.Spec.Replicas = operatorCR.Spec.Replicas
}

if operatorCR.Spec.Port != nil {
    deployment.Spec.Template.Spec.Containers[0].Ports[0].ContainerPort = *operatorCR.Spec.Port
}
```

### Giải thích

- Chỉ override khi user thật sự set field.
- Nếu user không set thì giữ default từ manifest.

### 5. Gắn owner reference

```go
ctrl.SetControllerReference(operatorCR, deployment, r.Scheme)
```

### Ý nghĩa

- CR trở thành owner của Deployment.
- Giúp garbage collection.
- Giúp controller watch resource phụ thuộc dễ hơn.

### 6. Create hoặc Update

```go
if create {
    err = r.Create(ctx, deployment)
} else {
    err = r.Update(ctx, deployment)
}
return ctrl.Result{}, err
```

### Ý nghĩa

- Nếu create/update fail thì trả lỗi để reconcile được retry.
- Nếu `err == nil` thì reconcile thành công.

### 7. Reconcile hoàn chỉnh trong chapter

```go
func (r *NginxOperatorReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    operatorCR := &operatorv1alpha1.NginxOperator{}

    err := r.Get(ctx, req.NamespacedName, operatorCR)
    if err != nil && errors.IsNotFound(err) {
        logger.Info("Operator resource object not found.")
        return ctrl.Result{}, nil
    } else if err != nil {
        logger.Error(err, "Error getting operator resource object")
        return ctrl.Result{}, err
    }

    deployment := &appsv1.Deployment{}
    create := false
    err = r.Get(ctx, req.NamespacedName, deployment)
    if err != nil && errors.IsNotFound(err) {
        create = true
        deployment = assets.GetDeploymentFromFile("manifests/nginx_deployment.yaml")
    } else if err != nil {
        logger.Error(err, "Error getting existing Nginx deployment.")
        return ctrl.Result{}, err
    }

    deployment.Namespace = req.Namespace
    deployment.Name = req.Name

    if operatorCR.Spec.Replicas != nil {
        deployment.Spec.Replicas = operatorCR.Spec.Replicas
    }

    if operatorCR.Spec.Port != nil {
        deployment.Spec.Template.Spec.Containers[0].Ports[0].ContainerPort = *operatorCR.Spec.Port
    }

    ctrl.SetControllerReference(operatorCR, deployment, r.Scheme)

    if create {
        err = r.Create(ctx, deployment)
    } else {
        err = r.Update(ctx, deployment)
    }

    return ctrl.Result{}, err
}
```

### Khung logic cần nhớ

1. Đọc CR.
2. Nếu CR không tồn tại thì thôi.
3. Đọc operand resource.
4. Nếu chưa có thì load default manifest.
5. So khớp desired state từ CR.
6. Create hoặc Update.

### 8. RBAC markers trong controller

```go
//+kubebuilder:rbac:groups=operator.example.com,resources=nginxoperators,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=operator.example.com,resources=nginxoperators/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=operator.example.com,resources=nginxoperators/finalizers,verbs=update
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
```

### Ý nghĩa

- đánh dấu quyền cần thiết ngay cạnh controller code
- `make manifests` sẽ generate RBAC tương ứng

### 9. Setup watch bằng builder pattern

```go
func (r *NginxOperatorReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&operatorv1alpha1.NginxOperator{}).
        Owns(&appsv1.Deployment{}).
        Complete(r)
}
```

### Ý nghĩa

- `For(...)`: watch CR chính.
- `Owns(...)`: watch cả Deployment thuộc CR.
- deployment bị xóa/chỉnh sai thì controller vẫn chạy lại.

### Điều cần ghi nhớ

- Tinh thần của controller-runtime: **event là tín hiệu**, còn `Reconcile()` phải làm việc theo kiểu **idempotent**.
- Không nên hiểu `Reconcile()` là hàm xử lý riêng cho create/update/delete event type.

---

## Troubleshooting

### 1. General Kubernetes resources

Nhiều lỗi nhìn như lỗi Operator SDK thực chất lại nằm ở tầng thấp hơn:

- Kubernetes API
- client libraries
- controller-runtime

Tài liệu tham khảo:

- Kubernetes docs
- Kubernetes API reference
- GitHub repositories của các dự án liên quan
- Kubernetes Slack

### Điều cần nhớ

Khi lỗi, hãy tự hỏi trước:

- mình đang lỗi ở **Go code**?
- lỗi ở **generated CRD/manifests**?
- lỗi ở **RBAC**?
- lỗi ở **API machinery**?
- hay lỗi ở **SDK CLI/generator**?

Nếu không phân lớp lỗi, bạn sẽ rất dễ tra sai tài liệu.

---

### 2. Operator SDK resources

- Operator SDK docs
- GitHub repo của Operator SDK
- Slack channels liên quan tới Operators

### Điều cần nhớ

Operator SDK chủ yếu cho bạn:

- CLI scaffolding
- workflow generate/build/deploy
- integration với controller-runtime/Kubebuilder

Nếu command CLI hoặc scaffold output có gì lạ, đây là nơi nên kiểm tra đầu tiên.

---

### 3. Kubebuilder resources

- Kubebuilder Book
- `controller-gen`
- GitHub repos của Kubebuilder/controller-tools
- Slack channel `#kubebuilder`

### Điều cần nhớ

Nếu lỗi liên quan tới:

- Kubebuilder markers
- CRD schema generation
- webhook generation
- controller-gen output

thì nguyên nhân thường nằm gần **Kubebuilder/controller-tools**, không phải riêng Operator SDK.

---

### Các link có thể hữu ích 

Nguồn chính thức nên đối chiếu:

- https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/
- https://sdk.operatorframework.io/docs/building-operators/golang/migration/
- https://book.kubebuilder.io/reference/generating-crd.html
- https://book.kubebuilder.io/reference/metrics
- https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md
- https://pkg.go.dev/embed

