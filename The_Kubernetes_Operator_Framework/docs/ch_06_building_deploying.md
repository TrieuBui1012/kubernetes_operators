# Building and Deploying Your Operator

## Technical requirements

### Kiến thức lý thuyết cần nhớ

Chuyển từ giai đoạn **viết Operator** sang giai đoạn **biên dịch, đóng gói và triển khai Operator vào cluster**. Trọng tâm là quy trình thủ công để:

* build binary cục bộ,
* build container image,
* đẩy image lên registry,
* deploy Operator vào cluster,
* kiểm tra operand,
* tích hợp metrics bằng Prometheus. 

Cách làm trong chương phù hợp với:

* phát triển cục bộ,
* thử nghiệm nhanh,
* môi trường proof-of-concept,
* disposable cluster.

Đây không phải là workflow production hoàn chỉnh 

### Yêu cầu môi trường

* Kết nối internet để kéo base image và đẩy image lên registry công khai.
* Quyền truy cập vào một Kubernetes cluster đang chạy.
* `kubectl` cài sẵn trên máy local.
* Docker cài cục bộ.
* Tài khoản Docker Hub hoặc registry công khai khác.
* Nên dùng cluster disposable như `kind` hoặc `minikube`. 

### Ghi nhớ

* Registry public không có bảo vệ truy cập sẽ khiến image của bạn ai cũng kéo được. Điều này chấp nhận được cho tutorial, nhưng không nên giữ nguyên cho production. 

---

## Building a container image

### Kiến thức lý thuyết cần nhớ

Kubernetes là nền tảng orchestration cho container, nên Operator cuối cùng cũng phải được chạy dưới dạng container. Operator SDK đã trừu tượng hóa phần lớn thao tác build/deploy bằng các target trong `Makefile`, giúp ta không phải tự gõ quá nhiều lệnh dài. 

### Lệnh xem các target có sẵn

```bash
$ make help

Usage:

  make <target>

General

  help             Display this help.

Development

  manifests        Generate WebhookConfiguration, ClusterRole 
and CustomResourceDefinition objects.

  generate         Generate code containing DeepCopy, 
DeepCopyInto, and DeepCopyObject method implementations.

  fmt              Run go fmt against code.

  vet              Run go vet against code.

  test             Run tests.

Build

  build            Build manager binary.

  run              Run a controller from your host.

  docker-build     Build docker image with the manager.

  docker-push      Push docker image with the manager.

Deployment

  install          Install CRDs into the K8s cluster specified 
in ~/.kube/config.

  uninstall        Uninstall CRDs from the K8s cluster 
specified in ~/.kube/config.

  deploy           Deploy controller to the K8s cluster 
specified in ~/.kube/config.

  undeploy         Undeploy controller from the K8s cluster 
specified in ~/.kube/config.

  controller-gen   Download controller-gen locally if 
necessary.

  kustomize        Download kustomize locally if necessary.

  envtest          Download envtest-setup locally if necessary.

  bundle           Generate bundle manifests and metadata, then 
validate generated files.

  bundle-build     Build the bundle image.

  bundle-push      Push the bundle image.

  opm              Download opm locally if necessary.

  catalog-build    Build a catalog image.

  catalog-push     Push a catalog image.
```

### Giải thích chi tiết các target quan trọng

`make manifests`
Sinh CRD, ClusterRole, WebhookConfiguration từ code/marker Kubebuilder.

`make generate`
Sinh các hàm hỗ trợ như `DeepCopy`, `DeepCopyInto`, `DeepCopyObject`.

`make fmt`
Format toàn bộ code Go.

`make vet`
Kiểm tra các lỗi tĩnh phổ biến trong code Go.

`make test`
Chạy test của project.

`make build`
Build binary cục bộ của Operator.

`make run`
Chạy controller từ máy local, không đóng gói container.

`make docker-build`
Build container image cho Operator.

`make docker-push`
Push image đã build lên registry.

`make deploy`
Deploy Operator vào cluster bằng các manifest scaffolded.

`make undeploy`
Gỡ Operator khỏi cluster.

Các target `bundle`, `bundle-build`, `bundle-push` là cho workflow OLM/bundle ở chương sau.

### Điều cần nhớ

Trong chương này, nhóm target quan trọng nhất là:

* `make build`
* `make docker-build`
* `make docker-push`
* `make deploy`
* `make undeploy`

---

## Building the Operator locally

### Kiến thức lý thuyết cần nhớ

Build local tạo ra **binary thực thi** của Operator. Đây là bước trung gian trước khi đóng gói binary đó vào container image. Trong scaffold mặc định, binary được build ra tên là `manager`. 

### Code từ Makefile

```make
build: generate fmt vet ## Build manager binary.

  go build -o bin/manager main.go
```

### Giải thích chi tiết

`build:`
Tên target.

`generate fmt vet`
Các dependency. Khi chạy `make build`, Make sẽ chạy các target này trước.

`go build -o bin/manager main.go`

* `go build`: biên dịch chương trình Go.
* `-o bin/manager`: ghi output thành file `bin/manager`.
* `main.go`: file entrypoint.

### Output khi chạy

```bash
$ make build

/home/nginx-operator/bin/controller-gen 
object:headerFile="hack/boilerplate.go.txt" paths="./..."

go fmt ./...

go vet ./...

go build -o bin/manager main.go
```

### Giải thích output

`controller-gen ...`
Sinh hoặc cập nhật code/manifests từ marker Kubebuilder.

`go fmt ./...`
Format toàn bộ package bên dưới thư mục hiện tại.

`go vet ./...`
Kiểm tra lỗi tĩnh.

`go build -o bin/manager main.go`
Build binary thực thi.

### Kết quả

Sau khi build thành công sẽ có file:

```bash
bin/manager
```

Đây là binary của Operator. Nó có thể chạy bằng `make run` hoặc chạy trực tiếp, nhưng vẫn chưa phải cách deploy chuẩn trong cluster; muốn chạy trong cluster thì cần đóng gói thành container image. 

---

## Building the Operator image with Docker

### Kiến thức lý thuyết cần nhớ

Build image là bước chuyển từ **binary cục bộ** sang **artifact có thể deploy trên Kubernetes**. Target `docker-build` chỉ là wrapper của lệnh `docker build`, nhưng được nối với workflow test/generate mặc định của project.

### Code từ Makefile

```make
docker-build: test ## Build docker image with the manager.

  docker build -t ${IMG} .
```

### Dockerfile của project

```dockerfile
# Build the manager binary

FROM golang:1.17 as builder

WORKDIR /workspace

# Copy the Go Modules manifests

COPY go.mod go.mod

COPY go.sum go.sum

# cache deps before building and copying source so that we 
don't need to re-download as much

# and so that source changes don't invalidate our downloaded 
layer

RUN go mod download

# Copy the go source

COPY cmd/main.go cmd/main.go

COPY api/ api/

COPY controllers/ controllers/

COPY assets/ assets

# Build

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -o 
manager main.go

# Use distroless as minimal base image to package the manager 
binary

# Refer to https://github.com/GoogleContainerTools/distroless 
for more details

FROM gcr.io/distroless/static:nonroot

WORKDIR /

COPY --from=builder /workspace/manager . 

USER 65532:65532

ENTRYPOINT ["/manager"]
```

### Giải thích chi tiết Dockerfile

`FROM golang:1.17 as builder`
Dùng image Go làm build stage.

`WORKDIR /workspace`
Đặt thư mục làm việc trong container build.

`COPY go.mod go.mod`
`COPY go.sum go.sum`
Copy metadata dependency trước để tận dụng cache.

`RUN go mod download`
Tải dependency của Go module.

`COPY main.go main.go`
`COPY api/ api/`
`COPY controllers/ controllers/`
`COPY assets/ assets`
Copy source code vào builder image.

**Điểm rất quan trọng của tutorial này** là dòng:

```dockerfile
COPY assets/ assets
```

vì project có thêm package `assets/`. Nếu thiếu dòng này, build trong Docker sẽ không thấy package `assets`.

`RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -o manager main.go`

* `CGO_ENABLED=0`: build binary tĩnh.
* `GOOS=linux`: build cho Linux.
* `GOARCH=amd64`: build cho amd64.
* `-a`: ép build lại package.
* `-o manager`: output là binary `manager`.

`FROM gcr.io/distroless/static:nonroot`
Runtime image tối giản.

`COPY --from=builder /workspace/manager .`
Copy binary từ build stage sang runtime stage.

`USER 65532:65532`
Chạy dưới user không phải root.

`ENTRYPOINT ["/manager"]`
Lệnh mặc định khi container khởi động.

### Build image

```bash
$ IMG=docker.io/sample/nginx-operator:v0.1 make docker-build

... 

docker build -t docker.io/sample/nginx-operator:v0.1 .

[+] Building 99.1s (18/18) FINISHED

 => [internal] load build definition from Dockerfile 0.0s

 => [builder  1/10] FROM docker.io/library/golang:1.17 21.0s

 => [builder  2/10] WORKDIR /workspace             2.3s

 => [builder  3/10] COPY go.mod go.mod             0.0s

 => [builder  4/10] COPY go.sum go.sum             0.0s

 => [builder  5/10] RUN go mod download            31.3s

 => [builder  6/10] COPY main.go main.go           0.0s

 => [builder  7/10] COPY api/ api/                 0.0s

 => [builder  8/10] COPY controllers/ controllers/ 0.0s

 => [builder  9/10] COPY assets/ assets            0.0s

 => [builder 10/10] RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 
go build -a -o manager main.go                     42.5s

 => [stage-1 2/3] COPY --from=builder /workspace/manager .

 => exporting to image                             0.2s

 => => exporting layers                            0.2s

 => => writing image sha256:dd6546d...b5ba118bdba4 0.0s

 => => naming to docker.io/sample/nginx-operator:v0.1
```

### Giải thích

`IMG=docker.io/sample/nginx-operator:v0.1`
Gán biến `IMG` ngay trên command line cho lần chạy này.

`make docker-build`
Gọi target build image.

### Kiểm tra image local

```bash
$ docker images

REPOSITORY              TAG       IMAGE ID       CREATED        
SIZE

sample/nginx-operator   v0.1      dd6546d2afb0   45 hours ago   
48.9MB
```

### Điều cần nhớ

* `IMG` là biến quyết định tên/tag image.
* Nếu không truyền `IMG`, scaffold cũ thường dùng mặc định `controller:latest`. 

---

## Deploying in a test cluster

### Kiến thức lý thuyết cần nhớ

Sau khi build xong image, cần làm cho cluster **kéo được image đó**. Chương dùng `kind` để tạo local cluster và Docker Hub làm public registry. Đây là workflow dễ nhất để thử nghiệm thủ công.

### Tạo cluster bằng kind

```bash
$ kind create cluster

Creating cluster "kind" ...

 Ensuring node image (kindest/node:v1.21.1)  

 Preparing nodes   

 Writing configuration  

 Starting control-plane  

 Installing CNI  

 Installing StorageClass  

 Set kubectl context to "kind-kind"

You can now use your cluster with

kubectl cluster-info --context kind-kind

Have a nice day!
```

### Kiểm tra cluster

```bash
$ kubectl cluster-info

Kubernetes master is running at https://127.0.0.1:56384

CoreDNS is running at https://127.0.0.1:56384/api/v1/
namespaces/kube-system/services/kube-dns:dns/proxy
```

### Giải thích

`kubectl cluster-info`
Hiển thị endpoint của API server và service hệ thống để xác nhận cluster hoạt động.

### Đẩy image lên registry

```bash
$ IMG=docker.io/sample/nginx-operator:v0.1 make docker-push 
docker push docker.io/sample/nginx-operator:v0.1

The push refers to repository [docker.io/sample/nginx-operator]

18452d09c8a6: Pushed 5b1fa8e3e100: Layer already exists 

v0.1: digest: sha256:5315a7092bd7d5af1dbb454c05c15c54131 
bd3ab78809ad1f3816f05dd467930 size: 739
```

### Giải thích

`docker login`
Cần chạy trước để xác thực với Docker Hub.

`IMG=... make docker-push`

* gán biến `IMG`,
* gọi target `docker-push`,
* target này thực chất gọi `docker push ${IMG}`.

### Cách tránh phải truyền IMG mỗi lần

```bash
$ export IMG=docker.io/sample/nginx-operator:v0.1 

$ make docker-push

docker push docker.io/sample/nginx-operator:v0.1

The push refers to repository [docker.io/sample/nginx-operator]

18452d09c8a6: Pushed 5b1fa8e3e100: Layer already exists 

v0.1: digest: sha256:5315a7092bd7d5af1dbb454c05c15c54131bd 
3ab78809ad1f3816f05dd467930 size: 739
```

### Ghi nhớ

Bạn cũng có thể kiểm tra image đã public hay chưa bằng:

```bash
docker pull <image>
```

nhưng không bắt buộc.

### Không muốn dùng Docker Hub

Sách nhắc một hướng khác:

```bash
kind load docker-image <image>
```

để nạp image local trực tiếp vào cluster kind thay vì push public.

### Deploy Operator vào cluster

```bash
$ make deploy

/Users/sample/nginx-operator/bin/controller-gen 
rbac:roleName=manager-role crd webhook paths="./..." 
output:crd:artifacts:config=config/crd/bases

cd config/manager && /Users/sample/nginx-operator/bin/kustomize 
edit set image controller=docker.io/sample/nginx-operator:v0.1

/Users/sample/nginx-operator/bin/kustomize build config/default 
| kubectl apply -f -

namespace/nginx-operator-system created

customresourcedefinition.apiextensions.k8s.io/nginxoperators.
operator.example.com created

serviceaccount/nginx-operator-controller-manager created

role.rbac.authorization.k8s.io/nginx-operator-leader-election-
role created

clusterrole.rbac.authorization.k8s.io/nginx-operator-manager-
role created

clusterrole.rbac.authorization.k8s.io/nginx-operator-metrics-
reader created

clusterrole.rbac.authorization.k8s.io/nginx-operator-proxy-role 
created

rolebinding.rbac.authorization.k8s.io/nginx-operator-leader-
election-rolebinding created

clusterrolebinding.rbac.authorization.k8s.io/nginx-operator-
manager-rolebinding created

clusterrolebinding.rbac.authorization.k8s.io/nginx-operator-
proxy-rolebinding created

configmap/nginx-operator-manager-config created

service/nginx-operator-controller-manager-metrics-service 
created

deployment.apps/nginx-operator-controller-manager created
```

### Giải thích

`controller-gen ...`
Regenerate RBAC/CRD/webhook.

`kustomize edit set image controller=...`
Thay image của controller trong manifest Kustomize bằng image bạn vừa build.

`kustomize build config/default | kubectl apply -f -`

* render manifest cuối,
* apply vào cluster qua stdin.

### Kiểm tra namespace

```bash
$ kubectl get namespaces

NAME                    STATUS   AGE

default                 Active   31m

kube-node-lease         Active   31m

kube-public             Active   31m

kube-system             Active   31m

local-path-storage      Active   31m

nginx-operator-system   Active   54s
```

### Kiểm tra resource của Operator

```bash
$ kubectl get all -n nginx-operator-system

NAME                                                     READY   
STATUS    pod/nginx-operator-controller-manager-6f5f66795d-
945pb   2/2     Running   

NAME                                                        
TYPE        service/nginx-operator-controller-manager-metrics-
service   ClusterIP   

NAME                                                READY   
UP-TO-DATE  deployment.apps/nginx-operator-controller-manager   
1/1     1            

NAME                                                           
DESIRED   replicaset.apps/nginx-operator-controller-manager-
6f5f66795d   1
```

### Tạo custom resource đầu tiên

```yaml
sample-cr.yaml:

apiVersion: operator.example.com/v1alpha1

kind: NginxOperator

metadata:

  name: cluster

  namespace: nginx-operator-system

spec:

  replicas: 1
```

### Giải thích

Đây là instance đầu tiên của CRD. Operator được thiết kế để chỉ reconcile khi thấy custom resource tồn tại.

### Tạo object

```bash
kubectl create -f sample-cr.yaml
```

### Kiểm tra pod sau khi tạo CR

```bash
$ kubectl get pods -n nginx-operator-system

NAME                                                 READY   
STATUS    cluster-7855777498-rcwdj                             
1/1     Running  

nginx-operator-controller-manager-6f5f66795d-hzb8n   2/2     
Running
```

### Sửa CR trực tiếp

```bash
$ kubectl edit nginxoperators/cluster -n nginx-operator-system
```

Ví dụ nội dung object khi chỉnh sửa:

```yaml
apiVersion: operator.example.com/v1alpha1

kind: NginxOperator

metadata:

  creationTimestamp: "2022-02-05T18:28:47Z"

  generation: 1

  name: cluster

  namespace: nginx-operator-system

  resourceVersion: "7116"

  uid: 66994aa7-e81b-4b18-8404-2976be3db1a7

spec:

  replicas: 2

status:

  conditions:

  - lastTransitionTime: "2022-02-05T18:28:47Z"

    message: operator successfully reconciling

    reason: OperatorSucceeded

    status: "False"
```

### Giải thích

`kubectl edit`

* mở editor local,
* chỉnh object trực tiếp trên API server,
* rất tiện cho dev/test.

Sau khi đổi `spec.replicas: 2`, pod nginx sẽ tăng lên:

```bash
$ kubectl get pods -n nginx-operator-system

NAME                                                 READY   
STATUS    cluster-7855777498-rcwdj                             
1/1     Running  

cluster-7855777498-kzs25                             1/1     
Running

nginx-operator-controller-manager-6f5f66795d-hzb8n   2/2     
Running
```

Nếu đổi `spec.replicas: 0` thì các pod operand sẽ bị xóa:

```bash
$ kubectl get pods -n nginx-operator-system

NAME                                                 READY   
STATUS    cluster-7855777498-rcwdj                             
1/1     Terminating  

cluster-7855777498-kzs25                             1/1     
Terminating

nginx-operator-controller-manager-6f5f66795d-hzb8n   2/2     
Running
```

---

## Pushing and testing changes

### Kiến thức lý thuyết cần nhớ

Phần này mô tả vòng lặp phát triển rất thực tế:

1. sửa project,
2. build lại,
3. deploy lại,
4. kiểm tra kết quả.

---

## Installing and configuring kube-prometheus

### Kiến thức lý thuyết cần nhớ

Metrics chỉ hữu ích khi có hệ thống scrape và hiển thị chúng. Ta có thể chọn **kube-prometheus-stack** để dựng trọn monitoring stack, thay vì cài Prometheus rời rạc.


---

## Redeploying the Operator with metrics

### Kiến thức lý thuyết cần nhớ

Prometheus đã biết **namespace nào** cần scrape, nhưng chưa biết **endpoint nào** trong namespace đó phải scrape. Muốn giải quyết, cần resource `ServiceMonitor`. 

### Sửa `config/default/kustomization.yaml`

```yaml
config/default/kustomization.yaml:

...

resources:

- ../crd

- ../rbac

- ../manager

# ...

# ... 

# [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'.                                    
- ../prometheus
```

### Sửa `config/prometheus/monitor.yaml`

```yaml
# Prometheus Monitor Service (Metrics)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    control-plane: controller-manager
    app.kubernetes.io/name: nginx-operator
    app.kubernetes.io/managed-by: kustomize
    release: kube-prometheus-stack # Thêm release name của kube-prometheus-stack
  name: controller-manager-metrics-monitor
  namespace: system
spec:
  endpoints:
    - path: /metrics
      port: https # Ensure this is the name of the port that exposes HTTPS metrics
      scheme: https
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      tlsConfig:
        # TODO(user): The option insecureSkipVerify: true is not recommended for production since it disables
        # certificate verification, exposing the system to potential man-in-the-middle attacks.
        # For production environments, it is recommended to use cert-manager for automatic TLS certificate management.
        # To apply this configuration, enable cert-manager and use the patch located at config/prometheus/servicemonitor_tls_patch.yaml,
        # which securely references the certificate from the 'metrics-server-cert' secret.
        insecureSkipVerify: true
  selector:
    matchLabels:
      control-plane: controller-manager
      app.kubernetes.io/name: nginx-operator
```

### Giải thích

* `../prometheus` là phần manifest bổ sung cho Prometheus integration,
* khi bỏ comment dòng này, Kustomize sẽ kéo thêm manifest liên quan đến metrics, trong đó quan trọng nhất là `ServiceMonitor`.

### Redeploy

```bash
make undeploy
make deploy
```

### Giải thích

`make undeploy`
Gỡ Operator hiện tại khỏi cluster.

`make deploy`
Deploy lại với cấu hình mới, lần này có thêm metrics integration.

Sau một lúc, metric `reconciles_total` sẽ xuất hiện trong Prometheus dashboard.

---

## Key takeaways

### Kiến thức cần chốt lại

Phần này không chỉ là “bật metrics”, mà còn là quy trình **redeploy theo hướng phát triển**:

* cài kube-prometheus vào project,
* cấu hình Prometheus scrape namespace của Operator,
* chỉnh file Kustomize để thêm dependency mới,
* dùng `make undeploy`,
* deploy lại để nhìn thấy metric mới. 

### Ghi nhớ

`make deploy` về lý thuyết đã đủ vì manifest Kubernetes có tính idempotent; chỉ resource mới mới được tạo thêm. Tuy nhiên, biết cách dùng `make undeploy` vẫn rất hữu ích khi cần dọn hẳn môi trường cũ. 

---

## Summary

### Tóm tắt kiến thức cốt lõi

Toàn bộ quy trình **biên dịch và cài đặt thủ công một Operator vào Kubernetes cluster**:

1. Dùng `make build` để build local binary.
2. Dùng `make docker-build` để build container image.
3. Dùng `make docker-push` để push image lên registry.
4. Dùng `kind` để dựng disposable test cluster.
5. Dùng `make deploy` để triển khai Operator vào cluster.
6. Tạo custom resource đầu tiên để kích hoạt reconcile logic.
7. Dùng `kubectl edit` để thay đổi spec và quan sát Operator điều chỉnh operand.
8. Cài `kube-prometheus` để có monitoring stack.
9. Bật `ServiceMonitor`/prometheus integration và redeploy Operator để nhìn thấy metric `reconciles_total`.
10. Debug các lỗi thường gặp quanh Make, kind, Docker và metrics.

### Ý nghĩa

Đây là workflow **dev/test thủ công**, rất tốt để:

* kiểm chứng code Operator,
* học cách artifact của Operator đi từ source code tới cluster,
* hiểu rõ từng lớp build/deploy/monitoring.

Nhưng đây chưa phải workflow production/distribution chuẩn. Tiếp theo ta sẽ sang nội dung:

* OLM,
* bundle image,
* OperatorHub,
* workflow phân phối và quản lý Operator chuẩn hơn.
