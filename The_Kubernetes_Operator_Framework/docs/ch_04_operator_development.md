# Developing an Operator with the Operator SDK

## 1. Ý chính cần ghi nhớ

### 1.1. Tổng quan

Hướng dẫn cách tạo một Operator cơ bản bằng **Operator SDK** theo hướng **Go-based operator**. Luồng chính của chương là:

1. Dùng `operator-sdk init` để scaffold project.
2. Dùng `operator-sdk create api` để tạo API type + controller skeleton.
3. Sinh CRD/manifests từ Go types bằng `make manifests`.
4. Thêm các tài nguyên phụ trợ, ví dụ Deployment của operand Nginx.
5. Điền logic cho hàm `Reconcile()` để đồng bộ trạng thái mong muốn với trạng thái thực tế trong cluster.

### 1.2. Mô hình tư duy cốt lõi

- **Operator là controller**: nó không chạy một vòng lặp kiểu `for {}` thủ công, mà được controller-runtime kích hoạt lại khi có event liên quan.
- **API trước, logic sau**: muốn viết logic cho Operator thì trước hết phải có kiểu dữ liệu cho CRD, vì đó là nơi user cấu hình Operator.
- **CRD sinh từ code, không viết tay**: Go structs + Kubebuilder markers là nguồn chân lý; file YAML CRD chỉ là đầu ra được generate.
- **Reconcile theo kiểu level-based**: mỗi lần reconcile phải đọc lại trạng thái hiện tại từ cluster và quyết định create/update gì.
- **Operator không nên hardcode namespace/resource name có sẵn**: tận dụng `req.NamespacedName` và object owner/reference để giảm giả định sai.

### 1.3. Mục tiêu chức năng của sample Operator trong chương

Sample Operator quản lý một deployment Nginx đơn giản, với 3 input chính qua CR:

- `port`: cổng expose trên container Nginx.
- `replicas`: số replica của deployment.
- `forceRedploy` (sách bị typo; đúng ra nên là `forceRedeploy`): field no-op để ép rollout lại operand.

Chức năng tối thiểu mà Operator đạt được ở chương này:

- Tạo operand deployment khi CR được tạo.
- Cập nhật deployment khi CR thay đổi.
- Theo dõi deployment để tự reconcile lại nếu deployment bị xóa/sai trạng thái.

---

## 2. Kiến trúc project sau khi scaffold

Sau khi chạy `operator-sdk init`, project mẫu có các thành phần chính:

- `main.go`: điểm vào, khởi tạo manager, health/readiness checks, đăng ký controller.
- `api/<version>/`: định nghĩa Go types cho CRD.
- `controllers/`: code reconcile.
- `config/`: manifests được generate hoặc dùng để deploy.
- `Dockerfile`: build image Operator.
- `Makefile`: gom các lệnh generate/build/test/deploy.

Điều cần nhớ: **`api/` định nghĩa giao diện người dùng của Operator**, còn **`controllers/` định nghĩa hành vi**.

---

## 3. Toàn bộ commands xuất hiện trong chapter 4 và giải thích

Bên dưới là toàn bộ các lệnh xuất hiện hoặc được dùng trực tiếp trong chương, kèm giải thích.

### 3.1. Tạo thư mục project

```bash
mkdir nginx-operator
cd nginx-operator
```

**Giải thích:**

- `mkdir nginx-operator`: tạo thư mục rỗng cho project Operator.
- `cd nginx-operator`: chuyển vào thư mục đó để mọi lệnh scaffold tiếp theo tạo file đúng chỗ.

---

### 3.2. Khởi tạo boilerplate project bằng Operator SDK

```bash
operator-sdk init --domain example.com --repo github.com/example/nginx-operator
```

**Ý nghĩa từng phần:**

- `operator-sdk init`: scaffold project Operator mới.
- `--domain example.com`: domain dùng để ghép thành API group, ví dụ `operator.example.com`.
- `--repo github.com/example/nginx-operator`: khai báo module/repository path cho project Go.

**Tác dụng:**

- Tạo `go.mod`, `main.go`, `Makefile`, `Dockerfile`, `config/`, `hack/`.
- Thiết lập project theo chuẩn controller-runtime/Kubebuilder mà Operator SDK hỗ trợ.

**Điểm cần nhớ:**

- `--domain` ảnh hưởng trực tiếp đến tên API group của CRD sau này.
- `--repo` phải khớp với module path nếu bạn dùng Git/go modules thật.

---

### 3.3. Xem cây file sau khi init

```bash
ls
```

**Giải thích:** dùng để kiểm tra các file scaffold đã được tạo.

Các file/thư mục quan trọng sách liệt kê:

- `config/`: YAML definitions và kustomize config.
- `hack/`: script/phụ trợ cho generate/verify.
- `.dockerignore`, `.gitignore`
- `Dockerfile`
- `Makefile`
- `PROJECT`
- `go.mod`, `go.sum`
- `main.go`

---

### 3.4. Tạo API + controller skeleton

```bash
operator-sdk create api --group operator --version v1alpha1 --kind NginxOperator --resource --controller
```

**Ý nghĩa:**

- `create api`: tạo Go types cho API và controller skeleton.
- `--group operator`: API group, ghép với domain thành `operator.example.com`.
- `--version v1alpha1`: version đầu tiên của API.
- `--kind NginxOperator`: tên Kind của custom resource.
- `--resource`: tạo resource/CRD liên quan.
- `--controller`: tạo controller cho resource đó.

**Kết quả sinh ra:**

- `api/v1alpha1/nginxoperator_types.go`
- `controllers/nginxoperator_controller.go`
- cập nhật `main.go` để đăng ký controller

**Điểm học được:**

- `Spec` nhận input từ user.
- `Status` trả output/trạng thái.
- CRD YAML sẽ được generate từ Go types chứ không viết trực tiếp.

---

### 3.5. Regenerate code sau khi sửa API types

```bash
make generate
```

**Giải thích:**

- Chạy generator để cập nhật code sinh tự động như `zz_generated.deepcopy.go`.
- Đây là bước nên chạy mỗi khi sửa struct/API markers trong `api/`.

**Vì sao quan trọng?**

- Nếu không generate lại, project có thể compile lỗi hoặc runtime hành xử sai do generated code không đồng bộ với type mới.

---

### 3.6. Sinh CRD và manifests

```bash
make manifests
```

**Giải thích:**

- Tạo hoặc cập nhật CRD YAML từ Go types.
- Đồng thời regenerate một số RBAC/webhook manifests dựa trên markers trong code.

**Đầu ra nổi bật:**

- `config/crd/bases/operator.example.com_nginxoperators.yaml`
- `config/rbac/role.yaml`

**Cần nhớ:**

- CRD nên coi là file generated.
- Không nên sửa schema generated bằng tay nếu không thật sự hiểu luồng generator.

---

### 3.7. Cài go-bindata (cách cũ, trước Go 1.16)

```bash
go get -u github.com/go-bindata/go-bindata/...
```

**Giải thích:** cài tool `go-bindata` để nhúng file tĩnh vào binary Go.

**Góc nhìn hiện đại:** cách này ngày nay chủ yếu chỉ có giá trị lịch sử; phần lớn project mới sẽ dùng `go:embed`.

---

### 3.8. Generate asset code bằng go-bindata

```bash
go-bindata -o assets/assets.go assets/...
```

**Giải thích:**

- đọc toàn bộ file bên dưới `assets/`
- generate ra `assets/assets.go`
- sau đó có thể gọi `assets.Asset(...)` trong code

---

### 3.9. Generate asset code bằng go-bindata nhưng bỏ qua file generated

```bash
go-bindata -o assets/assets.go -ignore=\assets\/assets\.go assets/...
```

**Giải thích:** tránh việc chạy lặp rồi lại nhúng chính file `assets.go` vừa generate vào output mới.

---

### 3.10. Chạy Operator locally

```bash
make run
```

**Ý nghĩa trong Makefile mặc định:** chạy controller trên máy local thay vì deploy vào cluster như một Pod.

**Khi nào hữu ích?**

- debug nhanh
- chạy thử reconcile logic với kubeconfig hiện tại

---

### 3.11. Build binary local

```bash
make build
```

**Bên dưới nó làm gì?**

- `make generate`
- `go fmt ./...`
- `go vet ./...`
- `go build -o bin/manager main.go`

**Kết quả:** sinh binary `bin/manager`.

---

### 3.12. Build riêng bằng Go

```bash
go build -o bin/manager main.go
```

**Giải thích:** compile binary manager trực tiếp.

**Trong chapter:** đây là lệnh cốt lõi mà `make build` gọi bên dưới.

---

### 3.13. Format code

```bash
go fmt ./...
```

**Giải thích:** format toàn bộ mã Go theo chuẩn công cụ của Go.

---

### 3.14. Vet code

```bash
go vet ./...
```

**Giải thích:** phát hiện lỗi/suspicious pattern phổ biến mà compiler chưa chắc bắt.

---

### 3.15. Chạy toàn bộ generator bằng make

```bash
make
```

**Giải thích trong ngữ cảnh chapter:** tác giả nói có thể chạy `make` để gọi toàn bộ generator thay vì chạy riêng từng target.

---

### 3.16. Tidy dependencies

```bash
go mod tidy
```

**Giải thích:** đồng bộ `go.mod` và `go.sum`, thêm dependency đang dùng và bỏ dependency thừa.

---

### 3.17. Cài controller-gen nếu cần

```bash
controller-gen -h
```

**Giải thích:** xem help của `controller-gen`, tool đứng sau nhiều bước generate manifests/CRD.

**Bài học:** `make manifests` thực chất chỉ là wrapper tiện lợi; khi cần tùy biến sâu, phải hiểu `controller-gen`.

---

## 4. Toàn bộ code/snippet quan trọng trong chapter 4 và giải thích

> Tôi giữ lại các snippet/code block quan trọng xuất hiện trong chapter và giải thích theo vai trò thực tế. Với các đoạn generated YAML hoặc boilerplate rất dài, tôi rút gọn phần lặp nhưng vẫn giữ nguyên trường/ý nghĩa mà chapter đang dạy.

### 4.1. Bộ ba type quan trọng của API

```go
// NginxOperatorSpec defines the desired state of NginxOperator
type NginxOperatorSpec struct {
    Foo string `json:"foo,omitempty"`
}

// NginxOperatorStatus defines the observed state of NginxOperator
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

**Giải thích:**

- `Spec`: trạng thái mong muốn, user ghi vào đây.
- `Status`: trạng thái quan sát được, controller cập nhật.
- `NginxOperator`: CR chính, bọc metadata + spec + status.
- `+kubebuilder:subresource:status`: cho phép update `/status` riêng.

**Ý nghĩa sâu hơn:** đây là chuẩn thiết kế phổ biến của Kubernetes API object, không riêng Operator.

---

### 4.2. Spec thật của sample Operator

```go
type NginxOperatorSpec struct {
    // Port is the port number to expose on the Nginx Pod
    Port *int32 `json:"port,omitempty"`

    // Replicas is the number of deployment replicas to scale
    Replicas *int32 `json:"replicas,omitempty"`

    // ForceRedploy is any string, modifying this field
    // instructs the Operator to redeploy the Operand
    ForceRedploy string `json:"forceRedploy,omitempty"`
}
```

**Giải thích từng field:**

- `Port *int32`: con trỏ để phân biệt `unset` với `0`.
- `Replicas *int32`: tương tự, user không set thì fallback về default manifest.
- `ForceRedploy string`: chỉ cần đổi giá trị là tạo event, giúp ép reconcile/rollout lại.

**Điểm cần rất chú ý:**

- Sách dùng tên `ForceRedploy` bị typo; về mặt học tập bạn nên hiểu ý đồ là `ForceRedeploy`.
- Nếu tự làm dự án thật, nên sửa tên field ngay từ đầu để tránh API xấu sống lâu.

---

### 4.3. Ví dụ field required/default bằng Kubebuilder markers

```go
// Port is the port number to expose on the Nginx Pod
// +kubebuilder:default=8080
// +kubebuilder:validation:Required
Port int `json:"port"`
```

**Giải thích:**

- `default=8080`: nếu user không điền thì API server/defaulting sẽ gán 8080.
- `validation:Required`: bắt buộc phải có field này.
- bỏ `omitempty`: field sẽ không còn optional nữa.

**Bài học:** validation/defaulting nên khai báo ở type-level thay vì tự viết kiểm tra rải rác trong reconcile.

---

### 4.4. CRD YAML sau khi generate

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

**Giải thích:**

- `group`: API group riêng.
- `plural/singular/kind`: cách resource xuất hiện trong API.
- `scope: Namespaced`: CR nằm trong namespace.
- `versions`: khai báo version API đang serve.
- `openAPIV3Schema`: validation schema generate từ Go types.
- `subresources.status`: bật status subresource.

**Bài học:** CRD YAML rất dài và chi tiết, nên coi nó là output thay vì nguồn chỉnh tay.

---

### 4.5. Role generate cho Operator CRD

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

**Giải thích:**

- Operator cần quyền trên CRD của chính nó để đọc spec và cập nhật status.
- `/status` và `/finalizers` là subresource riêng nên phải cấp quyền riêng.

**Góc nhìn thực tế:** nên cấp quyền tối thiểu cần thiết; sách giữ `create/delete` cho đơn giản, nhưng production nên xem lại theo nguyên tắc least privilege.

---

### 4.6. Tạo Deployment object trực tiếp trong Go

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

**Giải thích:**

- Cách này xây Kubernetes resource trực tiếp bằng Go struct.
- Ưu điểm: type-safe, IDE autocomplete tốt.
- Nhược điểm: verbose, khó đọc, khó đối chiếu với YAML quen thuộc của Kubernetes user.

**Bài học chương muốn truyền đạt:** có thể làm, nhưng với manifest lớn thì maintainability không đẹp bằng file YAML nhúng vào binary.

---

### 4.7. YAML Deployment của Nginx operand

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

**Giải thích:**

- Đây là operand manifest mẫu mà Operator sẽ tạo/update.
- `selector.matchLabels` phải khớp với `template.metadata.labels`.
- `replicas: 1` là default nếu CR không override.

**Lưu ý hiện đại:** không nên dùng `nginx:latest` trong hệ thống thật; nên pin tag hoặc digest.

---

### 4.8. Đọc asset bằng go-bindata

```go
import "github.com/sample/nginx-operator/assets"

asset, err := assets.Asset("nginx_deployment.yaml")
if err != nil {
    // process object not found error
}
```

**Giải thích:**

- `assets.Asset(...)` trả nội dung file đã được nhúng vào binary.
- Sau đó thường cần decode từ bytes sang object Kubernetes.

**Ý nghĩa học tập:** đây là mô hình nhúng file tĩnh kiểu cũ.

---

### 4.9. Nhúng file bằng `go:embed`

```go
package main

import (
    "embed"
)

//go:embed assets/nginx_deployment.yaml
var deployment embed.FS
```

**Giải thích:**

- `//go:embed`: chỉ thị compiler nhúng file vào binary.
- `embed.FS`: filesystem ảo để đọc file khi runtime.

**Đây là cách hiện đại hơn go-bindata** vì không cần tool ngoài để generate mã.

---

### 4.10. Decode YAML embedded thành Deployment object

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

**Giải thích:**

- `AddToScheme`: đăng ký type `Deployment` vào scheme.
- `ReadFile`: lấy bytes từ embedded FS.
- `runtime.Decode(...)`: parse bytes thành object Kubernetes.
- ép kiểu về `*appsv1.Deployment` để dùng tiếp.

**Bài học:** để decode YAML thành object typed, bạn cần scheme + decoder tương ứng API group/version.

---

### 4.11. Tách logic nhúng manifest vào package `assets`

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

**Giải thích:**

- Tách phần nhúng/đọc manifest ra package riêng giúp controller gọn hơn.
- `GetDeploymentFromFile(...)` trở thành helper cho controller.
- Cấu trúc này dễ mở rộng nếu sau này có thêm Service, ConfigMap, RBAC manifest...

---

### 4.12. Reconcile skeleton ban đầu

```go
func (r *NginxOperatorReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    _ = log.FromContext(ctx)

    // your logic here

    return ctrl.Result{}, nil
}
```

**Giải thích:**

- Đây là hàm cốt lõi của controller.
- `req` chứa `NamespacedName` của object kích hoạt reconcile.
- trả `ctrl.Result{}, nil` nghĩa là reconcile thành công, chưa yêu cầu requeue.

---

### 4.13. Lấy CR object của Operator trong Reconcile

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

**Giải thích logic:**

- `r.Get(...)`: đọc CR hiện tại.
- Nếu `IsNotFound`: CR chưa tồn tại hoặc vừa bị xóa; không cần retry.
- Nếu là lỗi khác: trả lỗi để controller-runtime requeue.

**Bài học cực quan trọng:** phân biệt lỗi “không tồn tại” với lỗi hạ tầng/thật sự.

---

### 4.14. Lấy Deployment hiện có, nếu chưa có thì đánh dấu create

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

**Giải thích:**

- Nếu Deployment chưa có, load manifest mặc định từ assets.
- Nếu có rồi, dùng object hiện tại để update.
- Cờ `create` quyết định gọi `Create()` hay `Update()` ở cuối hàm.

**Lưu ý:** ở đây ví dụ đang dùng một simplification là lookup Deployment bằng `req.NamespacedName`, tức ngầm giả định operand Deployment đi theo cùng tên/namespace với CR.

---

### 4.15. Áp CR settings lên Deployment

```go
if operatorCR.Spec.Replicas != nil {
    deployment.Spec.Replicas = operatorCR.Spec.Replicas
}

if operatorCR.Spec.Port != nil {
    deployment.Spec.Template.Spec.Containers[0].Ports[0].ContainerPort = *operatorCR.Spec.Port
}
```

**Giải thích:**

- Chỉ override khi user thật sự set field trong CR.
- Nếu không set, manifest default vẫn giữ nguyên.

**Bài học thiết kế:** optional field + manifest default = cách khá sạch để tách default khỏi reconcile logic.

---

### 4.16. Gắn owner reference

```go
ctrl.SetControllerReference(operatorCR, deployment, r.Scheme)
```

**Giải thích:**

- Đánh dấu CR là owner của Deployment.
- Giúp garbage collection và giúp controller dễ watch các object “thuộc về” CR này.

---

### 4.17. Create hoặc Update Deployment

```go
if create {
    err = r.Create(ctx, deployment)
} else {
    err = r.Update(ctx, deployment)
}
return ctrl.Result{}, err
```

**Giải thích:**

- Nếu create/update lỗi thì trả lỗi để reconcile được retry.
- Nếu `err == nil` thì reconcile thành công.

---

### 4.18. Hàm Reconcile hoàn chỉnh trong chapter

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

**Giải thích tổng thể:**

1. Đọc CR.
2. Nếu có lỗi nghiêm trọng khi đọc CR thì fail.
3. Đọc operand Deployment.
4. Nếu chưa có thì load manifest mặc định.
5. Ghi đè replica/port từ CR.
6. Gắn owner reference.
7. Create hoặc Update.

**Đây chính là khung xương của nhiều Operator đơn giản ngoài đời thật.**

**Lưu ý kỹ thuật:** đây vẫn là ví dụ học tập. Trong production, ngoài tính idempotent, bạn còn phải cân nhắc conflict khi `Update()`, patch strategy, defaulting/validation, và cách định danh operand resources chắc chắn hơn.

---

### 4.19. Kubebuilder RBAC markers trong controller

```go
//+kubebuilder:rbac:groups=operator.example.com,resources=nginxoperators,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=operator.example.com,resources=nginxoperators/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=operator.example.com,resources=nginxoperators/finalizers,verbs=update
//+kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
```

**Giải thích:**

- Đây là cách annotate quyền cần thiết ngay cạnh controller code.
- `make manifests` sẽ đọc markers này để generate RBAC YAML.

**Ưu điểm:**

- dễ review
- giảm việc quên cập nhật role khi controller bắt đầu dùng resource mới

---

### 4.20. Setup watch trong `SetupWithManager()`

```go
func (r *NginxOperatorReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&operatorv1alpha1.NginxOperator{}).
        Owns(&appsv1.Deployment{}).
        Complete(r)
}
```

**Giải thích:**

- `For(...)`: watch CR chính `NginxOperator`.
- `Owns(...)`: watch cả `Deployment` mà CR sở hữu.
- Khi deployment bị xóa/chỉnh sai, controller vẫn reconcile lại được.

**Bài học:** Operator không chỉ watch CR của nó mà còn watch các object phụ thuộc để tự-heal.

---

## 5. Walkthrough theo từng phần của chapter

### 5.1. Setting up your project

Mục này chỉ ra rằng boilerplate project không hề “trống” hoàn toàn. Nó đã có sẵn manager, health/readiness endpoints, Makefile targets và cấu trúc config. Nghĩa là Operator SDK không chỉ scaffold code, mà còn scaffold **workflow phát triển**.

### 5.2. Defining an API

Đây là phần quan trọng nhất của chương về mặt tư duy. Bạn không bắt đầu bằng cách viết reconcile trước, mà bắt đầu bằng **định nghĩa contract giữa user và controller**. Contract đó chính là CRD/Go types.

### 5.3. Adding resource manifests

Chương so sánh hai cách:

- viết resource trực tiếp bằng Go struct
- lưu resource ở YAML rồi nhúng vào binary

Thông điệp của chương: với Operator nhỏ và operand đơn giản, YAML embedded thường dễ bảo trì hơn.

### 5.4. Writing a control loop

Đây là chỗ chuyển từ “khai báo dữ liệu” sang “thực thi hành vi”. Chương xây dựng Reconcile theo kiểu rất chuẩn:

- read desired state
- inspect actual state
- if missing create
- if drifted update

### 5.5. Troubleshooting

Điểm hay của mục này là nó nhắc rằng Operator SDK thực ra đứng trên nhiều lớp:

- Kubernetes API machinery
- controller-runtime
- Kubebuilder/controller-gen
- Operator SDK CLI

Khi lỗi, phải hiểu mình đang lỗi ở lớp nào thì mới tra đúng tài liệu.

### 5.6. Điều chapter 4 chưa làm

Chapter 4 chủ yếu dừng ở mức:

- scaffold project
- định nghĩa API/CRD
- generate manifests
- viết control loop
- thiết lập watch và RBAC cơ bản

Nó **chưa** đi sâu vào việc build/push image, deploy end-to-end vào cluster, chạy custom resource trong cluster thật, hay các tính năng vận hành nâng cao như metrics, conditions, leader election. Các phần đó được nối tiếp ở chapter 5 và chapter 6.

---

## 6. Những điểm lỗi thời trong chapter và cách hiểu lại ở 2026

> Đây là phần “sửa trực tiếp” tinh thần của chapter để phù hợp thực tế hiện nay.

### 6.1. `go-bindata` không còn là lựa chọn mặc định

**Trong sách:** trình bày cả `go-bindata` lẫn `go:embed`.

**Cập nhật hiện đại:**

- Với Go hiện đại, **ưu tiên `go:embed`**.
- Chỉ cân nhắc `go-bindata` nếu bạn đang duy trì codebase cũ hoặc có ràng buộc build rất đặc biệt.

**Kết luận học tập:** cứ nhớ `go:embed` là hướng chuẩn cho project mới.

### 6.2. Không nên dùng image tag `latest`

**Trong sách:** dùng `nginx:latest` để minh họa.

**Hiện nay:**

- nên pin version cụ thể, ví dụ `nginx:1.27.x`
- tốt hơn nữa là pin digest trong môi trường production

**Lý do:** rollout tái lập được, tránh drift ngoài ý muốn.

### 6.3. Guidance về metrics mặc định trong sách đã cũ

**Trong sách:** nói metrics endpoint mặc định là `/metrics` trên port `8080`.

**Cập nhật 2026:**

- với scaffold mới, chi tiết metrics không nên ghi nhớ cứng theo `8080`
- nhiều project mới dùng **metrics được bảo vệ tốt hơn**, thường qua HTTPS/secure endpoint khi bật cấu hình tương ứng
- cách triển khai metrics hiện tại phụ thuộc scaffold version, `main.go`, kustomize config, và cấu hình manager của project

**Điều nên học đúng:**

- hãy nhớ rằng controller-runtime/Kubebuilder đã có hỗ trợ metrics sẵn
- nhưng **port, giao thức, và cách expose** phải kiểm tra trực tiếp từ scaffold/version thực tế của project
- đừng copy mù pattern cũ chỉ vì thấy trong sách

### 6.4. Một số lệnh/flow cũ của Operator SDK nên hiểu là lịch sử

**Trong sách:** lệnh `operator-sdk init` và `operator-sdk create api` phản ánh scaffold của giai đoạn 2022.

**Cập nhật:**

- khái niệm vẫn đúng
- nhưng flags, file layout, Makefile targets, và generated config có thể khác theo version mới
- các lệnh kiểu cũ như `operator-sdk generate crds`, `operator-sdk build`, `operator-sdk generate openapi` không còn là mental model nên học cho project mới; thay vào đó nên quen với `make manifests`, `make generate`, `make docker-build`, `make docker-push`

**Cách học đúng:**

- nhớ *luồng làm việc* hơn là nhớ 100% output file tree từng dòng
- luôn đối chiếu với docs/version CLI đang cài

### 6.5. Project layout rất cũ trong nhiều bài viết khác, nhưng không phải layout nên học cho project mới

Nếu bạn gặp tài liệu cũ nói đến các layout như `deploy/`, `build/Dockerfile`, `cmd/manager/main.go`, `pkg/apis`, `pkg/controllers`, hãy xem đó là **di sản lịch sử** của Operator SDK đời cũ.

Với cách học hiện nay, nên lấy **Kubebuilder-style scaffolding** làm chuẩn đọc hiểu, vì đó là layout mà Operator SDK hiện đại bám theo.

### 6.6. Reconcile kiểu “Update toàn object” cần cẩn thận hơn trong code thật

**Trong chapter:** controller lấy object rồi `Update()` khá trực tiếp.

**Trong thực tế:**

- update mù toàn object dễ tạo conflict, dễ đụng defaulted fields/managed fields
- nên cân nhắc pattern compare-and-patch hoặc create/update helpers an toàn hơn

**Điều cần giữ từ chapter:** hiểu nguyên lý trước; còn production-grade reconcile cần kỹ hơn ở phần mutate + patch strategy.

### 6.7. Tên field `ForceRedploy` là typo

**Trong sách:** `ForceRedploy`.

**Nên hiểu/sửa:** `ForceRedeploy`.

Nếu làm project thật, đừng copy typo này sang API thật vì API name xấu sẽ rất khó sửa về sau.

### 6.8. RBAC nên tối thiểu hơn ví dụ trong sách

Ví dụ cấp khá rộng để dễ học. Trong production:

- chỉ cấp verbs thực sự dùng
- nếu Operator không tự tạo/xóa CR của chính nó thì không nên giữ `create/delete` vô cớ

### 6.9. `panic()` chỉ phù hợp cho ví dụ học thuật ở một số chỗ

Trong code mẫu đọc embedded asset, chương dùng `panic()` để đơn giản hóa flow.

Trong dự án thật:

- nên trả lỗi rõ ràng hơn nếu có thể
- hoặc fail fast ở startup với log có ngữ cảnh đầy đủ

### 6.10. `make manifests` và `make generate` vẫn quan trọng, nhưng hãy xem chúng là wrapper

Điều cần học ở 2026 không phải chỉ là “gõ make manifests”, mà là hiểu:

- phía dưới nó gọi `controller-gen`
- file nào là generated
- marker nào ảnh hưởng output nào

Hiểu điều này giúp bạn debug nhanh hơn khi scaffold mới khác sách.

### 6.11. Nên ưu tiên controller builder patterns thay vì các kiểu watch wiring thủ công đời cũ

Khi học chapter này ở thời điểm hiện nay, hãy xem `SetupWithManager()` với:

- `For(...)`
- `Owns(...)`
- khi cần thì `Watches(...)`, predicates, map funcs

là cách tư duy chuẩn hơn nhiều so với các ví dụ controller-runtime đời cũ dùng watch wiring thủ công.

### 6.12. Hãy mô tả Reconcile là idempotent, không phải xử lý theo loại event

Một cách hiểu sai phổ biến là tưởng `Reconcile()` phải viết logic riêng cho create/update/delete event. Cách hiểu đúng hơn là:

- event chỉ là tín hiệu để controller chạy lại
- `Reconcile()` phải đọc trạng thái hiện tại trong cluster
- sau đó đưa cluster về desired state một cách **idempotent**

Đây là mental model rất quan trọng nếu bạn muốn đi từ ví dụ sách sang code production thật.

### 6.13. Nếu nói về CRD/webhook APIs, hãy ngầm hiểu chuẩn hiện tại là `v1`

Trong project mới, hãy mặc định nghĩ tới:

- `apiextensions.k8s.io/v1` cho CRD
- `admissionregistration.k8s.io/v1` cho webhook APIs

Các tài liệu hoặc snippets còn nói `v1beta1` chỉ nên được xem là tài liệu lịch sử.

---

## 6A. Nguồn chính thức nên đối chiếu khi học lại chapter 4 ở thời điểm hiện nay

- Operator SDK Go tutorial: https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/
- Operator SDK migration guide: https://sdk.operatorframework.io/docs/building-operators/golang/migration/
- Kubebuilder CRD generation: https://book.kubebuilder.io/reference/generating-crd.html
- Kubebuilder metrics reference: https://book.kubebuilder.io/reference/metrics
- controller-runtime FAQ: https://github.com/kubernetes-sigs/controller-runtime/blob/main/FAQ.md
- Go `embed` package: https://pkg.go.dev/embed

---

## 7. Checklist học nhanh sau khi đọc xong chapter 4

Nếu cần tự kiểm tra kiến thức, hãy chắc rằng bạn trả lời được các câu sau:

1. Vì sao phải định nghĩa API/Spec/Status trước khi viết reconcile?
2. `operator-sdk init` và `operator-sdk create api` khác nhau thế nào?
3. Vì sao CRD nên generate từ Go types thay vì viết tay?
4. Khi nào nên dùng field pointer trong `Spec`?
5. Vì sao `errors.IsNotFound(err)` thường không nên requeue như lỗi thật?
6. `For(...)` và `Owns(...)` trong `SetupWithManager()` khác nhau thế nào?
7. Vì sao `ownerReference` quan trọng?
8. Vì sao `go:embed` thường tốt hơn `go-bindata` ở project mới?
9. Vì sao không nên hardcode namespace/tên resource?
10. Vì sao ví dụ `Update()` trong chapter chỉ là bước khởi đầu, chưa phải production-grade reconcile hoàn chỉnh?

---

## 8. Tóm tắt một trang

- Chapter 4 là chương biến thiết kế Operator thành code chạy được.
- Trọng tâm là **Operator SDK + Kubebuilder workflow**.
- Bắt đầu bằng `init` → tạo `api` → sửa `Spec/Status` → `make manifests` → viết `Reconcile()`.
- `Spec` là input từ user; `Status` là output từ controller.
- Operator tạo và duy trì operand bằng cách đọc CR, so sánh trạng thái thực tế, rồi create/update resource liên quan.
- Watch không chỉ CR mà còn cả resource phụ thuộc (`Owns(...)`).
- Resource manifests có thể viết thẳng bằng Go hoặc nhúng YAML; với project mới nên ưu tiên `go:embed`.
- Nhiều chi tiết CLI/config trong sách mang tính 2022, nhưng nguyên lý kiến trúc và luồng làm việc vẫn còn nguyên giá trị.

---

## 9. Gợi ý đọc tiếp sau chapter 4

Sau chapter này, nên đọc chapter 5 với tâm thế sau:

- Chapter 4 trả lời: **làm sao để Operator chạy được?**
- Chapter 5 trả lời: **làm sao để Operator đáng dùng hơn trong production?**

Cụ thể, chapter 5 sẽ mở rộng thêm:

- status conditions
- metrics
- leader election
- health/readiness checks

Tức là đi từ Operator “biết reconcile” sang Operator “biết tự quan sát và vận hành an toàn hơn”.
