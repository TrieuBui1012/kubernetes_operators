#Preparing for Ongoing Maintenance of Your Operator

## Technical requirements

### Kiến thức lý thuyết cần nhớ

Chương 8 tập trung vào giai đoạn **sau khi Operator đã được phát hành**: tiếp tục phát triển, phát hành version mới, lập kế hoạch deprecation, bám theo chuẩn của Kubernetes, và theo dõi release timeline upstream. Ý chính của chương là: phần lớn công việc của một Operator thường diễn ra **sau bản phát hành đầu tiên**, nên maintainability và release discipline là cực kỳ quan trọng.  

---

## Releasing new versions of your Operator

### Kiến thức lý thuyết cần nhớ

Sau khi Operator được publish, công việc chưa kết thúc mà mới thực sự bắt đầu. Operator sẽ tiếp tục:

* thêm feature mới,
* sửa lỗi,
* thay đổi API,
* thích nghi với thay đổi từ upstream Kubernetes.

Vì vậy, phát hành version mới không chỉ là build lại image, mà còn liên quan tới:

* API versioning,
* conversion giữa các version,
* cập nhật CSV,
* đóng gói bundle mới,
* publish lại lên OperatorHub. 

---

## Adding an API version to your Operator

### Kiến thức lý thuyết cần nhớ

Một lý do rất phổ biến để phát hành version mới là **thay đổi config API** của Operator, tức API được chuyển thành CRD. Khi thay đổi đó là **breaking change**, cách làm chuẩn là:

1. tạo **API version mới**,
2. để cả version cũ và mới cùng tồn tại trong một thời gian,
3. thêm cơ chế **conversion** giữa các version,
4. sau đó mới nghĩ tới deprecation version cũ. 

---

### Command trong sách: khởi tạo API ban đầu

```bash
operator-sdk create api --group operator --version v1alpha1 \
--kind NginxOperator --resource --controller
```

### Giải thích chi tiết

* `operator-sdk create api`: scaffold một API mới cho Operator.
* `--group operator`: group của CRD.
* `--version v1alpha1`: tạo version đầu tiên của API.
* `--kind NginxOperator`: kind của custom resource.
* `--resource`: tạo resource/CRD tương ứng.
* `--controller`: tạo controller reconcile cho resource đó.

Lệnh này chính là nền tảng ban đầu của `NginxOperator` trong chương 4, và chương 8 dùng lại nó để giải thích logic thêm API version mới. 

---

### Ví dụ thay đổi API: từ `port` sang `ports`

#### Code cũ trong sách

```go
func (r *NginxOperatorReconciler) Reconcile(ctx context.
Context, req ctrl.Request) (ctrl.Result, error) {

  if operatorCR.Spec.Port != nil {

    deployment.Spec.Template.Spec.Containers[0].Ports[0].
ContainerPort = *operatorCR.Spec.Port

  }
```

#### Code mới trong sách

```go
func (r *NginxOperatorReconciler) Reconcile(ctx context.
Context, req ctrl.Request) (ctrl.Result, error) {

  if len(operatorCR.Spec.Ports) > 0 {

    deployment.Spec.Template.Spec.Containers[0].Ports = 
operatorCR.Spec.Ports

  }
```

### Giải thích chi tiết

**Đoạn cũ**

* `operatorCR.Spec.Port != nil`: kiểm tra custom resource có cấu hình port không.
* `deployment.Spec.Template.Spec.Containers[0].Ports[0].ContainerPort = *operatorCR.Spec.Port`: ghi giá trị đó vào port đầu tiên của container.

**Đoạn mới**

* `len(operatorCR.Spec.Ports) > 0`: kiểm tra danh sách port có phần tử không.
* `deployment.Spec.Template.Spec.Containers[0].Ports = operatorCR.Spec.Ports`: gán toàn bộ danh sách port cho container.

### Ý nghĩa kiến trúc

Đây là ví dụ điển hình của **breaking API change**:

* `v1alpha1`: chỉ hỗ trợ một port.
* `v1alpha2`: hỗ trợ nhiều port.

Lợi ích:

* linh hoạt hơn,
* tận dụng type chuẩn của Kubernetes (`v1.ContainerPort`),
* mở rộng được thêm thông tin như `Name`, `HostPort`, `Protocol`.

Chi phí:

* cần version mới,
* cần conversion,
* người dùng field cũ phải chuyển sang field mới. 

---

### Kiểu `NginxOperatorSpec` cũ trong `v1alpha1`

```go
// NginxOperatorSpec defines the desired state of NginxOperator

type NginxOperatorSpec struct {

   // Port is the port number to expose on the Nginx Pod

   Port *int32 `json:"port,omitempty"`

   // Replicas is the number of deployment replicas to scale

   Replicas *int32 `json:"replicas,omitempty"`

   // ForceRedploy is any string, modifying this field 
instructs

   // the Operator to redeploy the Operand

   ForceRedploy string `json:"forceRedploy,omitempty"`

}
```

### Kiểu `NginxOperatorSpec` mới trong `v1alpha2`

```go
// NginxOperatorSpec defines the desired state of NginxOperator

type NginxOperatorSpec struct {

   // Ports defines the ContainerPorts exposed on the Nginx Pod

   Ports []v1.ContainerPort `json:"ports,omitempty""`

   // Replicas is the number of deployment replicas to scale

   Replicas *int32 `json:"replicas,omitempty"`

   // ForceRedploy is any string, modifying this field 
instructs

   // the Operator to redeploy the Operand

   ForceRedploy string `json:"forceRedploy,omitempty"`

}
```

### Giải thích chi tiết

* `Port *int32` → `Ports []v1.ContainerPort`: từ scalar sang list object.
* `Replicas`: giữ nguyên.
* `ForceRedploy`: giữ nguyên mục đích, là field dùng để trigger redeploy.

### Lưu ý khi áp dụng thực tế

Trong code của sách có hai lỗi nhỏ:

* `json:"ports,omitempty""` có thừa một dấu `"` ở cuối.
* `ForceRedploy` có vẻ là typo của `ForceRedeploy`.

Nếu bạn áp dụng vào code thật thì nên sửa cho đúng cú pháp và nhất quán tên field. 

---

### Command trong sách: scaffold API version mới

```bash
$ operator-sdk create api --group operator --version v1alpha2 --kind NginxOperator --controller=false --make=false

Create Resources [y/n]

y

Writing kustomize manifests for you to edit...

Writing scaffold for you to edit...
```

### Giải thích chi tiết

* tạo thêm version `v1alpha2` cho cùng kind `NginxOperator`,
* chỉ tạo API/resource,
* **không tạo controller mới** vì ta vẫn muốn dùng một controller logic thống nhất.

Trả lời `y` ở prompt `Create Resources [y/n]`

---

### Interface conversion được sách trích dẫn

```go
package conversion

import "k8s.io/apimachinery/pkg/runtime"

// Convertible defines capability of a type to convertible i.e. 
it can be converted to/from a hub type.

type Convertible interface {

      runtime.Object

      ConvertTo(dst Hub) error

      ConvertFrom(src Hub) error

}

// Hub marks that a given type is the hub type for conversion. 
This means that

// all conversions will first convert to the hub type, then 
convert from the hub

// type to the destination type. All types besides the hub type 
should implement

// Convertible.

type Hub interface {

      runtime.Object

      Hub()

}
```

### Giải thích chi tiết

* `Hub`: version trung tâm.
* `Convertible`: những version còn lại phải implement 2 chiều chuyển đổi:

  * `ConvertTo(dst Hub)`
  * `ConvertFrom(src Hub)`

Đây là mô hình **hub-and-spoke conversion**:

* chọn một version làm hub,
* mọi version khác convert qua hub. 

### Đối chiếu với hiện tại

Kubebuilder Book hiện nay vẫn mô tả conversion theo đúng mô hình hub-and-spoke. ([Kubebuilder Book][5])

---

### Code: đánh dấu `v1alpha2` là Hub

```go
api/v1alpha2/nginxoperator_conversion.go:

package v1alpha2

// Hub defines v1alpha2 as the hub version

func (*NginxOperator) Hub() {}
```

### Giải thích

Method `Hub()` không cần logic bên trong; nó chỉ là dấu hiệu để controller-runtime/Kubebuilder biết đây là **hub version**. 

---

### Code: scaffold conversion ở `v1alpha1`

```go
api/v1alpha1/nginxoperator_conversion.go:

package v1alpha1

import (

   "github.com/sample/nginx-operator/api/v1alpha2"

   v1 "k8s.io/api/core/v1"

   "k8s.io/utils/pointer"

   "sigs.k8s.io/controller-runtime/pkg/conversion"

)

// ConvertTo converts v1alpha1 to v1alpha2

func (src *NginxOperator) ConvertTo(dst conversion.Hub) error {

   return nil

}

// ConvertFrom converts v1alpha2 to v1alpha1

func (dst *NginxOperator) ConvertFrom(src conversion.Hub) error 
{

   return nil

}
```

### Giải thích

* `v1alpha1` là spoke version.
* Vì vậy nó phải implement cả `ConvertTo` và `ConvertFrom`.



---

### `ConvertTo`

```go
// ConvertTo converts v1alpha1 to v1alpha2
func (src *NginxOperator) ConvertTo(dst conversion.Hub) error {
	objV1alpha2 := dst.(*v1alpha2.NginxOperator)
	objV1alpha2.ObjectMeta = src.ObjectMeta
	objV1alpha2.Status.Conditions = src.Status.Conditions

	if src.Spec.Replicas != nil {
		objV1alpha2.Spec.Replicas = src.Spec.Replicas
	}
	if len(src.Spec.ForceRedeploy) > 0 {
		objV1alpha2.Spec.ForceRedeploy = src.Spec.ForceRedeploy
	}
	if src.Spec.Port != nil {
		objV1alpha2.Spec.Ports = make([]v1.ContainerPort, 0, 1)
		objV1alpha2.Spec.Ports = append(objV1alpha2.Spec.Ports, v1.ContainerPort{ContainerPort: *src.Spec.Port})
	}

	return nil
}
```

### Giải thích chi tiết

* `objV1alpha2 := dst.(*v1alpha2.NginxOperator)`: ép kiểu hub object về đúng type đích.
* copy `ObjectMeta` và `Status.Conditions`: giữ metadata và status.
* `Replicas`, `ForceRedploy`: chuyển 1:1.
* `Port` → `Ports`: đưa port đơn thành một slice có đúng 1 phần tử.

Điểm quan trọng:

* chiều `v1alpha1` → `v1alpha2` này **không mất dữ liệu**. 

---

### Code: `ConvertFrom`

```go
// ConvertFrom converts v1alpha2 to v1alpha1
func (dst *NginxOperator) ConvertFrom(src conversion.Hub) error {
	objV1alpha2 := src.(*v1alpha2.NginxOperator)
	dst.ObjectMeta = objV1alpha2.ObjectMeta
	dst.Status.Conditions = objV1alpha2.Status.Conditions

	if objV1alpha2.Spec.Replicas != nil {
		dst.Spec.Replicas = objV1alpha2.Spec.Replicas
	}
	if len(objV1alpha2.Spec.ForceRedeploy) > 0 {
		dst.Spec.ForceRedeploy = objV1alpha2.Spec.ForceRedeploy
	}

	if len(objV1alpha2.Spec.Ports) > 0 {
		dst.Spec.Port = ptr.To(objV1alpha2.Spec.Ports[0].ContainerPort)
	}

	return nil
}
```

### Giải thích chi tiết

* `v1alpha2` → `v1alpha1` là chiều khó hơn.
* vì `v1alpha1` chỉ có một `Port`, nên code lấy **phần tử đầu tiên** trong `Ports`.

Điểm quan trọng:

* chiều này là **lossy conversion** nếu object `v1alpha2` có nhiều port.
* đây chính là lý do sách dùng ví dụ này để nói về deprecation và conversion policy sau đó. 

---

### Command: scaffold conversion webhook

```bash
$ operator-sdk create webhook --group operator --version v1alpha2 --kind NginxOperator --conversion --spoke v1alpha1 --make=false

Writing kustomize manifests for you to edit...

Writing scaffold for you to edit...

api/v1alpha2/nginxoperator_webhook.go

Webhook server has been set up for you.

You need to implement the conversion.Hub and conversion.
Convertible interfaces for your CRD types.

Update dependencies:

$ go mod tidy

Running make:

$ make generate

/Users/mdame/nginx-operator/bin/controller-gen 
object:headerFile="hack/boilerplate.go.txt" paths="./..."

Next: implement your new Webhook and generate the manifests 
with:

$ make manifests
```

### Giải thích chi tiết

* `operator-sdk create webhook --conversion`: scaffold conversion webhook.
* `--version v1alpha2`: version đang được scaffold.
* `--kind NginxOperator --group operator`: xác định resource.
* `--spoke`: liệt kê các version cần chuyển đổi, ngăn cách bởi dấu phẩy
* `go mod tidy`: dọn dependency.
* `make generate`: regenerate code.
* `make manifests`: regenerate manifest.

---

### Cập nhật project manifests để deploy webhook

#### `config/crd/kustomization.yaml`

```yaml
# This kustomization.yaml is not intended to be run by itself,
# since it depends on service name and namespace that are out of this kustomize package.
# It should be run by config/default
resources:
  - bases/operator.example.com_nginxoperators.yaml
# +kubebuilder:scaffold:crdkustomizeresource

patches:
  # [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix.
  # patches here are for enabling the conversion webhook for each CRD
  - path: patches/webhook_in_nginxoperators.yaml
# +kubebuilder:scaffold:crdkustomizewebhookpatch

# [WEBHOOK] To enable webhook, uncomment the following section
# the following config is for teaching kustomize how to do kustomization for CRDs.
configurations:
  - kustomizeconfig.yaml
```

#### `config/crd/patches/webhook_in_nginxoperators.yaml`

```yaml
# The following patch enables a conversion webhook for the CRD
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: nginxoperators.operator.example.com
spec:
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        service:
          namespace: system
          name: webhook-service
          path: /convert
      conversionReviewVersions:
        - v1
        - v1alpha1 # thêm cái này
        - v1alpha2 # thêm cái này
```

#### Uncomment `config/default/kustomization.yaml`

```yaml
...

# Adds namespace to all resources.
namespace: nginx-operator-system

# Value of this field is prepended to the
# names of all resources, e.g. a deployment named
# "wordpress" becomes "alices-wordpress".
# Note that it should also match with the prefix (text before '-') of the namespace
# field above.
namePrefix: nginx-operator-

# Labels to add to all resources and selectors.
#labels:
#- includeSelectors: true
#  pairs:
#    someName: someValue

resources:
  - ../crd
  - ../rbac
  - ../manager
  # [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in
  # crd/kustomization.yaml
  - ../webhook
  # [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER'. 'WEBHOOK' components are required.
  - ../certmanager
  # [PROMETHEUS] To enable prometheus monitor, uncomment all sections with 'PROMETHEUS'.
  - ../prometheus
  # [METRICS] Expose the controller manager metrics service.
  - metrics_service.yaml
# [NETWORK POLICY] Protect the /metrics endpoint and Webhook Server with NetworkPolicy.
# Only Pod(s) running a namespace labeled with 'metrics: enabled' will be able to gather the metrics.
# Only CR(s) which requires webhooks and are applied on namespaces labeled with 'webhooks: enabled' will
# be able to communicate with the Webhook Server.
#- ../network-policy

# Uncomment the patches line if you enable Metrics
patches:
  # [METRICS] The following patch will enable the metrics endpoint using HTTPS and the port :8443.
  # More info: https://book.kubebuilder.io/reference/metrics
  - path: manager_metrics_patch.yaml
    target:
      kind: Deployment

  # Uncomment the patches line if you enable Metrics and CertManager
  # [METRICS-WITH-CERTS] To enable metrics protected with certManager, uncomment the following line.
  # This patch will protect the metrics with certManager self-signed certs.
  #- path: cert_metrics_manager_patch.yaml
  #  target:
  #    kind: Deployment

  # [WEBHOOK] To enable webhook, uncomment all the sections with [WEBHOOK] prefix including the one in
  # crd/kustomization.yaml
  - path: manager_webhook_patch.yaml
    target:
      kind: Deployment

# [CERTMANAGER] To enable cert-manager, uncomment all sections with 'CERTMANAGER' prefix.
# Uncomment the following replacements to add the cert-manager CA injection annotations
replacements:
  # - source: # Uncomment the following block to enable certificates for metrics
  #     kind: Service
  #     version: v1
  #     name: controller-manager-metrics-service
  #     fieldPath: metadata.name
  #   targets:
  #     - select:
  #         kind: Certificate
  #         group: cert-manager.io
  #         version: v1
  #         name: metrics-certs
  #       fieldPaths:
  #         - spec.dnsNames.0
  #         - spec.dnsNames.1
  #       options:
  #         delimiter: '.'
  #         index: 0
  #         create: true
  #     - select: # Uncomment the following to set the Service name for TLS config in Prometheus ServiceMonitor
  #         kind: ServiceMonitor
  #         group: monitoring.coreos.com
  #         version: v1
  #         name: controller-manager-metrics-monitor
  #       fieldPaths:
  #         - spec.endpoints.0.tlsConfig.serverName
  #       options:
  #         delimiter: '.'
  #         index: 0
  #         create: true
  #
  # - source:
  #     kind: Service
  #     version: v1
  #     name: controller-manager-metrics-service
  #     fieldPath: metadata.namespace
  #   targets:
  #     - select:
  #         kind: Certificate
  #         group: cert-manager.io
  #         version: v1
  #         name: metrics-certs
  #       fieldPaths:
  #         - spec.dnsNames.0
  #         - spec.dnsNames.1
  #       options:
  #         delimiter: '.'
  #         index: 1
  #         create: true
  #     - select: # Uncomment the following to set the Service namespace for TLS in Prometheus ServiceMonitor
  #         kind: ServiceMonitor
  #         group: monitoring.coreos.com
  #         version: v1
  #         name: controller-manager-metrics-monitor
  #       fieldPaths:
  #         - spec.endpoints.0.tlsConfig.serverName
  #       options:
  #         delimiter: '.'
  #         index: 1
  #         create: true
  #
  - source: # Uncomment the following block if you have any webhook
      kind: Service
      version: v1
      name: webhook-service
      fieldPath: .metadata.name # Name of the service
    targets:
      - select:
          kind: Certificate
          group: cert-manager.io
          version: v1
          name: serving-cert
        fieldPaths:
          - .spec.dnsNames.0
          - .spec.dnsNames.1
        options:
          delimiter: "."
          index: 0
          create: true
  - source:
      kind: Service
      version: v1
      name: webhook-service
      fieldPath: .metadata.namespace # Namespace of the service
    targets:
      - select:
          kind: Certificate
          group: cert-manager.io
          version: v1
          name: serving-cert
        fieldPaths:
          - .spec.dnsNames.0
          - .spec.dnsNames.1
        options:
          delimiter: "."
          index: 1
          create: true
  #
  # - source: # Uncomment the following block if you have a ValidatingWebhook (--programmatic-validation)
  #     kind: Certificate
  #     group: cert-manager.io
  #     version: v1
  #     name: serving-cert # This name should match the one in certificate.yaml
  #     fieldPath: .metadata.namespace # Namespace of the certificate CR
  #   targets:
  #     - select:
  #         kind: ValidatingWebhookConfiguration
  #       fieldPaths:
  #         - .metadata.annotations.[cert-manager.io/inject-ca-from]
  #       options:
  #         delimiter: '/'
  #         index: 0
  #         create: true
  # - source:
  #     kind: Certificate
  #     group: cert-manager.io
  #     version: v1
  #     name: serving-cert
  #     fieldPath: .metadata.name
  #   targets:
  #     - select:
  #         kind: ValidatingWebhookConfiguration
  #       fieldPaths:
  #         - .metadata.annotations.[cert-manager.io/inject-ca-from]
  #       options:
  #         delimiter: '/'
  #         index: 1
  #         create: true
  #
  # - source: # Uncomment the following block if you have a DefaultingWebhook (--defaulting )
  #     kind: Certificate
  #     group: cert-manager.io
  #     version: v1
  #     name: serving-cert
  #     fieldPath: .metadata.namespace # Namespace of the certificate CR
  #   targets:
  #     - select:
  #         kind: MutatingWebhookConfiguration
  #       fieldPaths:
  #         - .metadata.annotations.[cert-manager.io/inject-ca-from]
  #       options:
  #         delimiter: '/'
  #         index: 0
  #         create: true
  # - source:
  #     kind: Certificate
  #     group: cert-manager.io
  #     version: v1
  #     name: serving-cert
  #     fieldPath: .metadata.name
  #   targets:
  #     - select:
  #         kind: MutatingWebhookConfiguration
  #       fieldPaths:
  #         - .metadata.annotations.[cert-manager.io/inject-ca-from]
  #       options:
  #         delimiter: '/'
  #         index: 1
  #         create: true
  #
  - source: # Uncomment the following block if you have a ConversionWebhook (--conversion)
      kind: Certificate
      group: cert-manager.io
      version: v1
      name: serving-cert
      fieldPath: .metadata.namespace # Namespace of the certificate CR
    targets: # Do not remove or uncomment the following scaffold marker; required to generate code for target CRD.
      - select:
          kind: CustomResourceDefinition
          name: nginxoperators.operator.example.com
        fieldPaths:
          - .metadata.annotations.[cert-manager.io/inject-ca-from]
        options:
          delimiter: "/"
          index: 0
          create: true
  # +kubebuilder:scaffold:crdkustomizecainjectionns
  - source:
      kind: Certificate
      group: cert-manager.io
      version: v1
      name: serving-cert
      fieldPath: .metadata.name
    targets: # Do not remove or uncomment the following scaffold marker; required to generate code for target CRD.
      - select:
          kind: CustomResourceDefinition
          name: nginxoperators.operator.example.com
        fieldPaths:
          - .metadata.annotations.[cert-manager.io/inject-ca-from]
        options:
          delimiter: "/"
          index: 1
          create: true
# +kubebuilder:scaffold:crdkustomizecainjectionname

```

#### `config/webhook/kustomization.yaml`

```yaml
resources:
  # - manifests.yaml
  - service.yaml

configurations:
  - kustomizeconfig.yaml
```

### Giải thích chi tiết

Các file này làm những việc sau:

* bật patch để thêm conversion webhook vào CRD,
* bật webhook service,
* bật cert-manager để cấp certificate cho webhook server,
* nối certificate/service metadata vào manifest bằng Kustomize vars hoặc replacements,
* expose endpoint `/convert` cho API server gọi khi cần chuyển version.

---

### Cập nhật controller để dùng `v1alpha2`

```go
controllers/nginxoperator_controller.go:

func (*NginxOperatorReconciler) Reconcile(…) {

  if len(operatorCR.Spec.Ports) > 0 {

   deployment.Spec.Template.Spec.Containers[0].Ports = 
operatorCR.Spec.Ports

  }

}
```

### Giải thích

Sau khi đưa API mới vào, logic controller cũng phải chuyển sang đọc field `Ports` thay vì `Port`. Nếu quên bước này, code có thể vẫn compile được nhưng reconcile thực tế sẽ không khớp với schema mới. 

---

### Build, deploy và test API version mới

#### Build/deploy Operator

```bash
$ export IMG=docker.io/mdame/nginx-operator:v0.0.2

$ make docker-build docker-push

$ make deploy
```

### Giải thích chi tiết

* `export IMG=...`: đặt image tag cho Operator.
* `make docker-build`: build container image.
* `make docker-push`: push image lên registry.
* `make deploy`: apply manifests để triển khai Operator lên cluster.



---

#### Tạo custom resource theo version cũ

```yaml
sample-cr.yaml:

apiVersion: operator.example.com/v1alpha1

kind: NginxOperator

metadata:

  name: cluster

  namespace: nginx-operator-system

spec:

  replicas: 1

  port: 8080
```

```bash
$ kubectl apply -f sample-cr.yaml
```

### Giải thích

* dù Operator đã hỗ trợ `v1alpha2`, ta vẫn tạo object bằng `v1alpha1` để kiểm tra conversion.
* `kubectl apply -f sample-cr.yaml`: tạo custom resource instance.



---

#### Lấy object dưới version mới

```bash
$ kubectl get -o yaml nginxoperators/cluster -n nginx-operator-system

apiVersion: operator.example.com/v1alpha2

kind: NginxOperator

metadata:

  ...

  name: cluster

  namespace: nginx-operator-system

  resourceVersion: "9032"

  uid: c22f6e2f-58a5-4b27-be6e-90fd231833e2

spec:

  ports:

  - containerPort: 8080

    protocol: TCP

  replicas: 1

...
```

### Giải thích

* object được đọc ra ở representation `v1alpha2`,
* `port: 8080` từ bản cũ đã được convert thành:

  * `ports:`
  * `- containerPort: 8080`



---

#### Lấy object dưới version cũ để kiểm tra convert ngược

```bash
$ kubectl get -o yaml nginxoperators.v1alpha1.operator.example.
com/cluster -n nginx-operator-system

apiVersion: operator.example.com/v1alpha1

kind: NginxOperator

metadata:

  name: cluster

  namespace: nginx-operator-system

  resourceVersion: "9032"

  uid: c22f6e2f-58a5-4b27-be6e-90fd231833e2

spec:

  port: 8080

  replicas: 1
```

### Giải thích

Lệnh này buộc API server trả object theo version `v1alpha1`, nhờ đó bạn kiểm tra được chiều conversion ngược có hoạt động không. 

### Đối chiếu với hiện tại

Cách test conversion bằng `kubectl get` theo từng API version cụ thể vẫn hoàn toàn phù hợp với CRD versioning hiện nay. ([Kubernetes][2])

---

## Updating the Operator CSV version

### Kiến thức lý thuyết cần nhớ

CSV là nơi OLM và OperatorHub đọc metadata của Operator:

* version hiện tại,
* version nào thay thế version nào,
* upgrade path,
* channel.

Nếu bạn phát hành version mới mà không cập nhật CSV đúng cách, OLM sẽ không hiểu đúng đồ thị nâng cấp. 

---

### Cập nhật `replaces` trong base CSV

```yaml
config/manifests/bases/nginx-operator.clusterserviceversion.yaml:

apiVersion: operators.coreos.com/v1alpha1

kind: ClusterServiceVersion

metadata:

  annotations:

    alm-examples: '[]'

    capabilities: Basic Install

  name: nginx-operator.v0.0.0

  namespace: placeholder

spec:

  ...

  replaces: nginx-operator.v0.0.1
```

### Giải thích chi tiết

* `kind: ClusterServiceVersion`: CSV resource.
* `metadata.name`: tên CSV.
* `spec.replaces`: khai báo CSV mới sẽ thay thế CSV cũ nào.

Ví dụ trong sách:

* version mới đang build là `v0.0.2`
* nên phải khai báo rằng nó thay `v0.0.1`.



### Đối chiếu với hiện tại

`replaces` vẫn là một cơ chế hợp lệ trong OLM/community operators để mô tả upgrade edge. Community Operators hiện nay còn tài liệu hóa rõ `replaces-mode` như một chiến lược update graph. ([Operator Hub][7])

---

### Cập nhật `VERSION` trong Makefile

```makefile
# VERSION defines the project version for the bundle. 

       

# Update this value when you upgrade the version of your 
project. 

# To re-generate a bundle for another specific version without 
changing the standard setup, you can:  

# - use the VERSION as arg of the bundle target (e.g make 
bundle VERSION=0.0.2) 

# - use environment variables to overwrite this value (e.g 
export VERSION=0.0.2) 

VERSION ?= 0.0.2
```

### Giải thích chi tiết

* `VERSION ?= 0.0.2`: đặt default bundle version là `0.0.2`.
* vì dùng `?=`, bạn vẫn có thể override bằng:

  * command line: `make bundle VERSION=0.0.2`
  * env var: `export VERSION=0.0.2`



---

### Generate bundle mới

```bash
$ make bundle IMG=docker.io/sample/nginx-operator:v0.0.2
```

### Giải thích chi tiết

Lệnh này:

* generate bundle cho version mới,
* cập nhật CSV trong `bundle/manifests/...`,
* mang theo cả thông tin API version mới và conversion webhook nếu bạn đã thêm ở phần trước.

### Đối chiếu với hiện tại

`make bundle` vẫn là workflow chuẩn để tạo `bundle/` chứa manifests và metadata cho Operator SDK/OLM. Tài liệu hiện nay cũng ghi rõ `bundle-build` và `bundle-push` vẫn là các bước kế tiếp nếu bạn đóng gói bundle image. ([Operator SDK][8])

---

### Build và chạy bundle image

```bash
$ export BUNDLE_IMG=docker.io/sample/nginx-bundle:v0.0.2

$ make bundle-build bundle-push

$ operator-sdk run bundle docker.io/same/nginx-bundle:v0.0.2
```

### Giải thích chi tiết

* `export BUNDLE_IMG=...`: đặt tên image cho bundle.
* `make bundle-build`: build bundle image.
* `make bundle-push`: push bundle image.
* `operator-sdk run bundle ...`: yêu cầu OLM cài Operator từ bundle image đó.

---

## Releasing a new version on OperatorHub

### Kiến thức lý thuyết cần nhớ

Với OperatorHub/community operators, phát hành version mới về cơ bản vẫn là:

1. tạo thư mục version mới,
2. copy bundle mới vào đó,
3. mở pull request,
4. chờ CI pass và merge.

### Cách làm theo sách

Nếu version đầu là:

* `nginx-operator/0.0.1`

thì version mới sẽ là:

* `nginx-operator/0.0.2`

Sau đó:

* copy nội dung thư mục `bundle/` mới sinh vào thư mục version mới,
* commit,
* push lên fork,
* mở PR vào repo chính `community-operators`.

---

## Planning for deprecation and backward compatibility

### Revisiting Operator design

### Kiến thức lý thuyết cần nhớ

Sách quay lại ba guideline đã nêu từ sớm:

* **Start small**
* **Iterate effectively**
* **Deprecate gracefully**

Đây là ba nguyên tắc giúp hạn chế số lần phải tạo breaking change và giúp release sau này bền vững hơn. 

---

### Starting small

Nginx Operator ban đầu chỉ expose ba field:

* `port`
* `replicas`
* `force redeploy`

Đây là một thiết kế kiểu **MVP**:

* đủ để chạy,
* dễ hiểu,
* ít gánh nặng support,
* ít “knobs” cho người dùng hơn. 

### Ý chính cần nhớ

Nếu CRD ngay từ đầu đã quá nhiều field:

* người dùng khó hiểu,
* maintainers khó giữ ổn định,
* chi phí support tăng,
* technical debt tích tụ nhanh hơn.

---

### Iterating effectively

Thay đổi từ `port` sang `ports` là ví dụ điển hình:

* đổi schema làm tăng độ phức tạp,
* nhưng đổi lại tận dụng type native `v1.ContainerPort`,
* support nhiều port,
* bám type ổn định từ upstream hơn.

Điểm mấu chốt:

* chỉ nên làm breaking change khi lợi ích đủ lớn,
* và phải cân nhắc chi phí chuyển đổi cho người dùng.

---

### Deprecating gracefully

Khi bỏ field `port` và đưa vào `ports`, người dùng version cũ sẽ phải chuyển đổi. Nếu không có hỗ trợ, họ có thể bị mất dữ liệu hoặc bị gãy cấu hình. Vì thế sách thêm **conversion webhook** để tự động map từ field cũ sang field mới. 

### Ý chính cần nhớ

* graceful deprecation không có nghĩa là “không bao giờ bỏ field cũ”,
* mà là:

  * có version mới,
  * có conversion,
  * có thời gian chuyển tiếp,
  * có thông báo rõ cho người dùng.

---

## Complying with Kubernetes standards for changes

### Kiến thức lý thuyết cần nhớ

Kubernetes có **deprecation policy** riêng cho API lifecycle. Operator bên thứ ba không bắt buộc phải tuân thủ hoàn toàn, nhưng hiểu và bám theo policy này sẽ giúp:

* đồng bộ kỳ vọng với hệ sinh thái Kubernetes,
* dễ chuẩn bị hơn trước thay đổi upstream,
* tránh tự tạo ra incompatibility không cần thiết.

### Đối chiếu với hiện tại

Kubernetes hiện nay vẫn duy trì **Deprecation Policy** và **Deprecated API Migration Guide** như nguồn chuẩn cho việc theo dõi API bị deprecate và bị remove ở từng release. ([Kubernetes][11])

---

### Removing APIs

**Rule #1 trong sách**:
API elements chỉ được remove bằng cách **tăng version của API group**.

### Ý nghĩa

* không xóa field khỏi version hiện có,
* muốn bỏ field phải tạo version mới.

Điều này giúp người dùng version cũ không bị gãy workflow đột ngột. 

### Điều cần nhớ

* **xóa field** → version mới
* **thêm field mới** → thường có thể làm trong version hiện tại nếu không phá client cũ

---

### API conversion

**Rule #2 trong sách**:
API objects phải có thể **round-trip giữa các version mà không mất dữ liệu**.

Sách thừa nhận ví dụ nginx Operator **không đạt trọn vẹn rule này**, vì nhiều `ports` của `v1alpha2` không thể chuyển nguyên vẹn về `v1alpha1` chỉ có một `port`. 

### Hai hướng sách gợi ý để cải thiện

1. thêm `ports` vào cả `v1alpha1` và `v1alpha2`
2. hoặc lưu dữ liệu thừa vào annotation để khi convert ngược vẫn phục hồi được

### Điều cần nhớ

* goal lý tưởng là **lossless conversion**
* ví dụ của sách chỉ là **minh họa**, không phải thiết kế tốt nhất cho production

---

### API lifetime

Sách chia stability level thành:

* alpha
* beta
* GA

Với ví dụ nginx Operator:

* API đang ở **alpha**
* alpha về kỹ thuật không có cam kết support dài hạn
* beta thì sách nêu timeline hỗ trợ là **dài hơn giữa 9 tháng hoặc 3 release**
* GA thì được xem là mức ổn định cao nhất. 

---

## Aligning with the Kubernetes release timeline

### Kiến thức lý thuyết cần nhớ

Sách khuyên maintainer của Operator nên theo dõi **release timeline của Kubernetes** vì điều đó giúp:

* biết khi nào API mới lên GA,
* biết khi nào API cũ sắp bị remove,
* biết khi nào nên thử nâng dependency,
* giảm độ trễ giữa release của Kubernetes và release tương thích của Operator. 

---

### Overview of a Kubernetes release

Theo sách:

* release cycle dài khoảng **15 tuần**
* mục tiêu là **3 release mỗi năm** kể từ Kubernetes 1.22. 

### Ý chính cần nhớ

Bạn không nhất thiết phải phát hành Operator theo đúng cùng cadence đó, nhưng bạn nên biết:

* upstream đang ở pha nào,
* feature nào sẽ vào release nào,
* API deprecation nào sắp đến.

---

### Start of release

Đây là mốc bắt đầu chính thức của release cycle kế tiếp. Nó là mốc gốc để tính ra các freeze khác. 

---

### Enhancements Freeze

Đây là mốc để quyết định:

* enhancement/KEP nào được vào release hiện tại,
* enhancement nào phải dời sang release sau.

### Ý chính cần nhớ

Nếu Operator của bạn phụ thuộc vào một upstream feature mới, đây là một deadline rất đáng quan tâm, vì nó cho biết feature đó có thực sự kịp vào release không. 

---

### Call for Exceptions

Nếu enhancement không kịp freeze, maintainer có thể xin exception. Exception chỉ được duyệt khi mức rủi ro chấp nhận được và không gây nguy hiểm cho độ ổn định release. 

---

### Code Freeze

Đây là mốc mà gần như mọi thay đổi code mới phải dừng lại. Sau đó chủ yếu chỉ còn bug fix quan trọng. Sách gợi ý đây là thời điểm hợp lý để downstream thử cập nhật dependency theo RC hoặc artifact gần release. 

### Ý chính cần nhớ

Đừng để Operator của bạn tụt quá xa khỏi upstream, nếu không technical debt sẽ phình lên rất nhanh.

---

### Test Freeze

Sau Code Freeze, vẫn còn một khoảng ngắn để chỉnh test, rồi tới Test Freeze thì việc thay đổi test cũng bị siết chặt hơn rất nhiều. 

---

### GA release / Code Thaw

Đây là ngày release chính thức của minor version mới. Sau đó:

* Kubernetes gắn tag release,
* các dependency chuẩn như:

  * `k8s.io/api`
  * `k8s.io/apimachinery`
  * `k8s.io/client-go`
    cũng được cập nhật tương ứng,
* nhánh chính mở lại để nhận thay đổi cho release tiếp theo.

---

### Retrospective

Đây là giai đoạn nhìn lại release vừa kết thúc:

* review vấn đề,
* ghi nhận điểm tốt,
* rút kinh nghiệm cho các release sau.

Với downstream maintainer, đây là nơi hữu ích để hiểu release machinery của Kubernetes đang tiến hóa theo hướng nào. 

---

## Working with the Kubernetes community

### Kiến thức lý thuyết cần nhớ

Sách nhấn mạnh rằng tất cả các policy và timeline ở trên không phải là những “mệnh lệnh từ trên trời rơi xuống”, mà là kết quả của một quy trình **mở** do cộng đồng Kubernetes xây dựng. Vì vậy, nếu Operator của bạn phụ thuộc vào Kubernetes, thì việc tham gia cộng đồng upstream là điều rất có ích cho chính sản phẩm của bạn. 

### Những gì sách khuyến nghị

* vào Kubernetes Slack,
* theo dõi GitHub repositories,
* tham gia SIG meetings,
* hỗ trợ người khác,
* chủ động tham gia thảo luận upstream. 

---

## Summary

### Kiến thức lý thuyết cần nhớ

Đây là chương nói về điều còn quan trọng hơn cả bản phát hành đầu tiên của Operator: **bản phát hành tiếp theo**. Những ý chính cần nhớ của toàn chương là:

1. **Phát hành version mới của Operator** không chỉ là build image mới, mà còn có thể bao gồm:

   * thêm API version,
   * conversion giữa các version,
   * cập nhật CSV,
   * build bundle mới,
   * publish lại trên OperatorHub.

2. **Deprecation và backward compatibility** nên được tính ngay từ thiết kế:

   * bắt đầu nhỏ,
   * lặp lại có chọn lọc,
   * deprecate có lộ trình.

3. **Kubernetes deprecation policy** là một template rất tốt để học theo:

   * bỏ field thì tăng API version,
   * conversion nên càng ít mất dữ liệu càng tốt,
   * support timeline phụ thuộc vào stability level.

4. **Bám theo release timeline của Kubernetes** giúp bạn:

   * biết khi nào nên nâng dependency,
   * chuẩn bị trước deprecation upstream,
   * tránh technical debt tích tụ. 

5. **Làm việc với cộng đồng Kubernetes** là một phần thực tế của việc phát triển Operator lâu dài. Nó giúp bạn cập nhật thông tin sớm hơn và có tiếng nói trong những thay đổi có thể ảnh hưởng trực tiếp đến sản phẩm của mình. 



