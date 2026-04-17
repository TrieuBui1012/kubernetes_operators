# Installing and Running Operators with the Operator Lifecycle Manager

## Mục tiêu

Chuyển từ một Operator đang phát triển trong môi trường local sang một Operator có thể được **đóng gói, cài đặt, phân phối và dùng bởi người khác**. Hai trụ cột chính được đi sâu là:

* **OLM (Operator Lifecycle Manager)**: quản lý cài đặt, nâng cấp và vòng đời Operator trong cluster.
* **OperatorHub**: catalog công khai để phân phối và khám phá Operators. 

Ý chính:

* Manual deploy ở Chương 6 là đủ cho local dev/test.
* Muốn đưa Operator ra ngoài cho người dùng, cần workflow chuẩn hơn:

  * đóng gói thành **bundle**
  * để **OLM** cài
  * public qua **OperatorHub**. 

---

## Technical requirements

### Kiến thức lý thuyết cần nhớ

Tiếp tục dùng **nginx Operator** đã xây ở các chương trước. Nó giả định bạn đã có:

* một Kubernetes cluster đang chạy
* quyền admin trên cluster
* `kubectl`
* `operator-sdk`
* Docker
* một public container registry
* tài khoản GitHub nếu muốn submit lên OperatorHub. 

### Những gì cần có

* Access tới Kubernetes cluster
* `kubectl` trên máy local
* `operator-sdk` trên máy local
* Docker đang chạy
* GitHub account và quen với fork / pull request nếu muốn submit Operator. 

### Ghi nhớ

* Nên dùng **disposable cluster** như `kind` hoặc `minikube` khi làm lab.
* Gắn chặt với workflow build/push/deploy của Chương 6 và API/controller đã làm ở Chương 4–5. 

---

## Understanding the OLM

### Kiến thức lý thuyết cần nhớ

**OLM** là công cụ để **cài đặt** và **quản lý** Operators trong cluster. Nó giúp:

* cài Operator
* quản lý upgrade Operator
* làm cho Operator đã cài “discoverable” với người dùng cluster
* enforce dependencies
* tránh xung đột API giữa các Operators. 

Một điểm rất quan trọng: dù nghe có vẻ là một hệ thống phức tạp, OLM thực chất vẫn chỉ là **một tập resource manifests** được cài vào cluster, gồm:

* Pods / Deployments
* CRDs
* namespaces
* ServiceAccounts
* RoleBindings. 

### Ghi nhớ

* OLM là **trụ cột thứ hai** của Operator Framework, sau Operator SDK.
* OLM không bắt buộc về mặt kỹ thuật để chạy Operator.
* Nhưng OLM là cách **chuẩn và tiện hơn** để deploy/manage Operators ở quy mô thực tế. 

---

## Installing the OLM in a Kubernetes cluster

### Kiến thức lý thuyết cần nhớ

Trước khi cài Operator bằng OLM, phải cài **chính OLM** vào cluster trước. Sách khuyến nghị dùng cluster disposable để dễ reset nếu có lỗi. Với `kind`, có thể xóa cluster cũ để bắt đầu sạch. 

### Commands/codes trong mục

#### 1. Xóa cluster kind cũ nếu muốn bắt đầu sạch

```bash
kind delete cluster
```

#### Giải thích

* `kind`: công cụ tạo Kubernetes cluster bằng Docker
* `delete cluster`: xóa cluster hiện tại
* Mục đích: reset lab environment

---

#### 2. Cài OLM

```bash
$ operator-sdk olm install

INFO[0000] Fetching CRDs for version "latest"           
INFO[0000] Fetching resources for resolved version "latest" 
...
INFO[0027]   Deployment "olm/packageserver" successfully rolled 
out INFO[0028] Successfully installed OLM version "latest"
```

#### Giải thích chi tiết

* `operator-sdk`: CLI của Operator SDK
* `olm install`: cài OLM lên cluster hiện tại

Lệnh này làm các việc:

* tải CRDs của OLM
* tải các manifests liên quan
* apply vào cluster
* chờ các Deployment của OLM rollout thành công. 

---

#### 3. Kiểm tra tài nguyên OLM trong namespace `olm`

```bash
$ kubectl get all -n olm

NAME                                    READY   STATUS    
RESTARTS   AGE
pod/catalog-operator-5c4997c789-xr986   1/1     Running   0          
4m35s
pod/olm-operator-6d46969488-nsrcl       1/1     Running   0          
4m35s
pod/operatorhubio-catalog-h97sx         1/1     Running   0          
4m27s
pod/packageserver-69649dc65b-qppvg      1/1     Running   0          
4m26s
pod/packageserver-69649dc65b-xc2fr      1/1     Running   0          
4m26s

NAME                            TYPE        CLUSTER-IP      
EXTERNAL-IP   
service/operatorhubio-catalog   ClusterIP   10.96.253.116   
<none>        
service/packageserver-service   ClusterIP   10.96.208.29    
<none>        

NAME                               READY   UP-TO-DATE   
AVAILABLE   AGE
deployment.apps/catalog-operator   1/1     1            1           
4m35s
deployment.apps/olm-operator       1/1     1            1           
4m35s
deployment.apps/packageserver      2/2     2            2           
4m26s

NAME                                          DESIRED   CURRENT   
READY   
replicaset.apps/catalog-operator-5c4997c789   1         1         
1       
replicaset.apps/olm-operator-6d46969488       1         1         
1       
replicaset.apps/packageserver-69649dc65b      2         2         
2
```

Điểm cần nhớ:

* OLM tạo nhiều Pod/Deployment để thực hiện chức năng của nó
* sách nhấn mạnh có **5 Pods** trong namespace này phối hợp với nhau để cung cấp các chức năng cốt lõi của OLM, như:

  * theo dõi subscriptions
  * theo dõi custom resources đại diện cho việc cài Operator. 

### Ghi nhớ

* Sau khi cài OLM, namespace `olm` là nơi đầu tiên cần kiểm tra.
* Nếu các Pod chính của OLM chưa `Running`, chưa nên tiếp tục phần bundle/install Operator.

---

## Interacting with the OLM

### Kiến thức lý thuyết cần nhớ

Ngoài `olm install`, Operator SDK còn có các lệnh khác để làm việc với OLM:

* `operator-sdk olm uninstall`
* `operator-sdk olm status`

Chúng giúp:

* gỡ OLM
* kiểm tra tình trạng cài đặt của OLM
* debug khi có resource OLM bị thiếu/hỏng. 

### Commands/codes trong mục

#### 1. Kiểm tra trạng thái OLM

```bash
$ operator-sdk olm status

INFO[0000] Fetching CRDs for version "v0.20.0"          
INFO[0000] Fetching resources for resolved version "v0.20.0" 
INFO[0002] Successfully got OLM status for version "v0.20.0" 

NAME                                            NAMESPACE    
KIND                        STATUS

operatorgroups.operators.coreos.com                          
CustomResourceDefinition    Installed
operatorconditions.operators.coreos.com                      
CustomResourceDefinition    Installed
olmconfigs.operators.coreos.com                              
CustomResourceDefinition    Installed
installplans.operators.coreos.com                            
CustomResourceDefinition    Installed
clusterserviceversions.operators.coreos.com                  
CustomResourceDefinition    Installed
olm-operator-binding-olm                                     
ClusterRoleBinding          Installed
operatorhubio-catalog                           olm          
CatalogSource               Installed
olm-operators                                   olm          
OperatorGroup               Installed
aggregate-olm-view                                           
ClusterRole                 Installed
catalog-operator                                olm          
Deployment                  Installed
cluster                                                      
OLMConfig                   Installed
operators.operators.coreos.com                               
CustomResourceDefinition    Installed
olm-operator                                    olm          
Deployment                  Installed
subscriptions.operators.coreos.com                           
CustomResourceDefinition    Installed
aggregate-olm-edit                                           
ClusterRole                 Installed
olm                                                          
Namespace                   Installed
global-operators                                operators    
OperatorGroup               Installed
operators                                                    
Namespace                   Installed
packageserver                                   olm          
ClusterServiceVersion       Installed
olm-operator-serviceaccount                     olm          
ServiceAccount              Installed
catalogsources.operators.coreos.com                          
CustomResourceDefinition    Installed
system:controller:operator-lifecycle-manager                 
ClusterRole                 Installed
```

#### Giải thích chi tiết

* `olm status`: kiểm tra xem toàn bộ các CRD/resource cốt lõi của OLM có hiện diện đúng hay không
* Output cho biết:

  * resource nào đã được cài
  * loại resource
  * namespace nếu có
  * trạng thái `Installed`. 

---

#### 2. Mô phỏng lỗi bằng cách xóa CRD `OperatorGroup`

```bash
kubectl delete crd/operatorgroups.operators.coreos.com
```

Sau đó chạy lại:

```bash
operator-sdk olm status
```

Khi đó `olm status` sẽ báo lỗi kiểu:

* `no matches for kind "OperatorGroup" in version "operators.coreos.com/v1"`

Điều này cho thấy OLM đang thiếu CRD cần thiết. 

---

#### 3. Gỡ OLM

```bash
operator-sdk olm uninstall
```

#### Giải thích

* gỡ OLM khỏi cluster
* dùng khi muốn reset/cài lại OLM do lỗi

### Ghi nhớ quan trọng

* **Gỡ OLM không gỡ các Operators** mà OLM đã cài.
* Đây là hành vi có chủ đích để tránh mất dữ liệu.
* Muốn gỡ Operator, phải dùng workflow cleanup riêng, không chỉ gỡ OLM. 

---

## Running your Operator

### Kiến thức lý thuyết cần nhớ

Ở Chương 6, Operator đã được build/deploy theo kiểu thủ công. Nhưng để OLM hiểu và cài Operator, cần chuyển Operator sang định dạng **bundle**.

**Bundle** là gói metadata/manifests chuẩn cho OLM, mô tả:

* Operator
* CRD
* dependencies
* cách cài Operator. 

### Ghi nhớ

* Manual deploy và OLM deploy là hai workflow khác nhau.
* Để OLM cài Operator, cần:

  1. generate bundle
  2. build bundle image
  3. push bundle image
  4. `operator-sdk run bundle ...`

---

## Generating an Operator's bundle

### Kiến thức lý thuyết cần nhớ

Bundle là tập hợp nhiều manifest mô tả Operator theo định dạng OLM hiểu được. Sau khi tạo bundle manifests, ta có thể build chúng thành **bundle image** để OLM pull và cài vào cluster. 

### Command chính

```bash
$ make bundle

/Users/mdame/nginx-operator/bin/controller-gen 
rbac:roleName=manager-role crd webhook paths="./..." 
output:crd:artifacts:config=config/crd/bases

operator-sdk generate kustomize manifests -q

Display name for the operator (required): 
> Nginx Operator

Description for the operator (required): 
> Operator for managing a basic Nginx deployment

Provider's name for the operator (required): 
> MyCompany

Any relevant URL for the provider name (optional): 
> http://mycompany.example

Comma-separated list of keywords for your operator (required): 
> nginx,tutorial

Comma-separated list of maintainers and their emails (e.g. 
'name1:email1, name2:email2') (required): 
> Mike Dame:mike@mycompany.example

cd config/manager && /Users/mdame/nginx-operator/bin/kustomize 
edit set image controller=controller:latest

/Users/mdame/nginx-operator/bin/kustomize build config/
manifests | operator-sdk generate bundle -q --overwrite 
--version 0.0.1  

INFO[0000] Creating bundle.Dockerfile                   
INFO[0000] Creating bundle/metadata/annotations.yaml    
INFO[0000] Bundle metadata generated successfully       

operator-sdk bundle validate ./bundle

INFO[0000] All validation tests have completed successfully
```

### Giải thích chi tiết

`make bundle` thực hiện một chuỗi bước:

1. **Sinh CRD/RBAC/webhook manifests** bằng `controller-gen`
2. **Sinh Kustomize manifests**
3. Hỏi bạn về metadata cần đưa vào CSV:

   * display name
   * description
   * provider
   * provider URL
   * keywords
   * maintainers
4. Dùng `kustomize build`
5. Pipe kết quả sang `operator-sdk generate bundle`
6. Validate bundle bằng `operator-sdk bundle validate ./bundle`. 

### Metadata bạn nhập có ý nghĩa gì?

* **Display name**: tên hiển thị của Operator
* **Description**: mô tả chức năng
* **Provider’s name**: đơn vị/cá nhân phát triển
* **Provider URL**: link tới website / tổ chức / dự án
* **Keywords**: phục vụ tìm kiếm/phân loại
* **Maintainers + email**: thông tin liên hệ hỗ trợ. 

### Ghi nhớ

* Bundle generation gắn chặt với **CSV**.
* CSV là file trung tâm chứa metadata để OLM và OperatorHub hiểu Operator. 

---

## Exploring the bundle files

### Kiến thức lý thuyết cần nhớ

Sau `make bundle`, project có:

* thư mục `bundle/`
* file `bundle.Dockerfile` ở root. 

Trong `bundle/` có 3 thư mục con:

* `tests/`
* `metadata/`
* `manifests/` 

### Ý nghĩa từng phần

#### `tests/`

Chứa cấu hình cho **scorecard tests** để validate bundle.

#### `metadata/`

Chứa `annotations.yaml`, cung cấp cho OLM thông tin về:

* version
* dependencies

Lưu ý:

* annotations ở đây phải khớp với labels trong `bundle.Dockerfile`. 

#### `manifests/`

Chứa:

* CRD
* metrics resources (nếu có)
* và đặc biệt là **CSV**, file chứa phần lớn metadata của Operator. 

---

### Các phần quan trọng trong CSV

Sách dùng file:
`nginx-operator.clusterserviceversion.yaml`

#### 1. Metadata + sample CR + capability level

```yaml
apiVersion: operators.coreos.com/v1alpha1

kind: ClusterServiceVersion

metadata:

  annotations:

    alm-examples: |-

      [

        {

          "apiVersion": "operator.example.com/v1alpha1",

          "kind": "NginxOperator",

          "metadata": {

            "name": "nginxoperator-sample"

          },

          "spec": null

        }

      ]

    capabilities: Basic Install

    operators.operatorframework.io/builder: operator-
sdk-v1.17.0
```

#### Giải thích

* `alm-examples`: ví dụ custom resource để người dùng tham khảo
* `capabilities`: capability level của Operator
* `builder`: tool đã dùng để build bundle. 

---

#### 2. CRD mà Operator sở hữu

```yaml
spec:

  apiservicedefinitions: {}

  customresourcedefinitions:

    owned:

    - description: NginxOperator is the Schema for the 
nginxoperators API

      displayName: Nginx Operator

      kind: NginxOperator

      name: nginxoperators.operator.example.com

      version: v1alpha1
```

#### Giải thích

* `owned`: danh sách CRD thuộc về Operator
* cho OLM biết Operator này quản lý loại custom resource nào

---

#### 3. Thông tin hiển thị và version

```yaml
  description: Operator for managing a basic Nginx 
deployment

  displayName: Nginx Operator

  icon:

  - base64data: ""

    mediatype: ""

  keywords:

  - nginx

  - tutorial

  links:

  - name: Nginx Operator

    url: https://nginx-operator.domain

  maintainers:

  - email: mike@mycompany.example

    name: Mike Dame

  maturity: alpha

  provider:

    name: MyCompany

    url: http://mycompany.example

  version: 0.0.1
```

#### Giải thích

* `displayName`, `description`: thông tin hiển thị
* `icon`: icon của Operator
* `keywords`: từ khóa tìm kiếm
* `maintainers`: người chịu trách nhiệm
* `provider`: đơn vị phát triển
* `version`: version hiện tại. 

### Ghi nhớ

* CSV là file **quan trọng nhất** trong bundle.
* OLM dùng CSV để cài đặt.
* OperatorHub dùng CSV để hiển thị metadata cho người dùng. 

---

## Building a bundle image

### Kiến thức lý thuyết cần nhớ

Sau khi có bundle manifests, cần build chúng thành **bundle image** để OLM có thể dùng. Việc này dùng `bundle.Dockerfile`. 

### Command/code trong mục

#### 1. Xem `bundle.Dockerfile`

```bash
$ cat bundle.Dockerfile 

FROM scratch

# Core bundle labels.

LABEL operators.operatorframework.io.bundle.mediatype.v1=registry+v1
LABEL operators.operatorframework.io.bundle.manifests.v1=manifests/
LABEL operators.operatorframework.io.bundle.metadata.v1=metadata/
LABEL operators.operatorframework.io.bundle.package.v1=nginx-operator
LABEL operators.operatorframework.io.bundle.channels.v1=alpha
LABEL operators.operatorframework.io.metrics.builder=operator-sdk-v1.17.0
LABEL operators.operatorframework.io.metrics.mediatype.v1=metrics+v1
LABEL operators.operatorframework.io.metrics.project_layout=go.kubebuilder.io/v3

# Labels for testing.

LABEL operators.operatorframework.io.test.mediatype.v1=scorecard+v1
LABEL operators.operatorframework.io.test.config.v1=tests/scorecard/

# Copy files to locations specified by labels.

COPY bundle/manifests /manifests/
COPY bundle/metadata /metadata/
COPY bundle/tests/scorecard /tests/scorecard/
```

#### Giải thích chi tiết

* `FROM scratch`: image rỗng tối thiểu
* `LABEL ...`: metadata để OLM/Operator tooling hiểu image
* `COPY ...`: copy bundle files vào image đúng vị trí. 

---

#### 2. Build bundle image

```bash
$ make bundle-build

docker build -f bundle.Dockerfile -t example.com/nginx-operator-bundle:v0.0.1 .

[+] Building 0.4s (7/7) FINISHED
...
 => => naming to example.com/nginx-operator-bundle:v0.0.1
```

#### Giải thích

* `make bundle-build`: target Makefile để build bundle image
* thực chất gọi:

```bash
docker build -f bundle.Dockerfile -t example.com/nginx-operator-bundle:v0.0.1 .
```

* `-f bundle.Dockerfile`: chỉ định Dockerfile
* `-t`: gắn tag image
* `.`: build context. 

---

#### 3. Kiểm tra image local

```bash
$ docker images

REPOSITORY                         TAG       IMAGE ID       CREATED         SIZE
example.com/nginx-operator-bundle  v0.0.1    6b4bf32edd5d   5 hours ago     17.5kB
```

#### Giải thích

* `docker images`: liệt kê image local
* dùng để xác nhận bundle image đã build xong. 

---

#### 4. Đặt tên bundle image tùy chỉnh

```bash
$ BUNDLE_IMG=docker.io/myregistry/nginx-bundle:v0.0.1 make bundle-build
```

#### Giải thích

* `BUNDLE_IMG=...`: ghi đè biến tên image bundle cho lần chạy này
* giúp build đúng image name/tag mong muốn. 

### Ghi nhớ

* `IMG` thường dùng cho operator image
* `BUNDLE_IMG` dùng cho **bundle image**
* đây là hai image khác nhau

---

## Pushing a bundle image

### Kiến thức lý thuyết cần nhớ

Giống operator image, bundle image cũng phải được push lên registry mà cluster/OLM truy cập được. Nếu bundle image chỉ tồn tại local, OLM sẽ không thể dùng nó để cài Operator. 

### Command/code trong mục

```bash
$ make bundle-push

/Library/Developer/CommandLineTools/usr/bin/make docker-push IMG=docker.io/mdame/nginx-bundle:v0.0.1

docker push docker.io/mdame/nginx-bundle:v0.0.1

The push refers to repository [docker.io/mdame/nginx-bundle]

79c3f933fff3: Pushed 
93e60c892495: Pushed 
dd3276fbf1b2: Pushed 

v0.0.1: digest: sha256:f6938300b1b8b5a2ce127273e2e48443ad3ef2e558cbcf260d9b03dd00d2f230 size: 939
```

### Giải thích chi tiết

* `make bundle-push`: target push bundle image
* thực chất gọi `docker push ...`
* giá trị image được lấy từ `BUNDLE_IMG`
* output cuối cùng trả về **digest** của image đã push. 

### Ghi nhớ

* luôn kiểm tra image path/tag trước khi push
* dùng biến `BUNDLE_IMG` giúp giảm lỗi push nhầm repository

---

## Deploying an Operator bundle with the OLM

### Kiến thức lý thuyết cần nhớ

Khi đã có bundle image trên registry, OLM có thể cài Operator chỉ từ bundle đó. Đây là bước biến bundle thành Operator đang chạy trong cluster. 

### Command/code trong mục

#### 1. Chạy bundle bằng OLM

```bash
$ operator-sdk run bundle docker.io/mdame/nginx-bundle:v0.0.1

INFO[0013] Successfully created registry pod: docker-io-mdame-nginx-bundle-v0-0-1 
INFO[0013] Created CatalogSource: nginx-operator-catalog 
INFO[0013] OperatorGroup "operator-sdk-og" created      
INFO[0013] Created Subscription: nginx-operator-v0-0-1-sub 
INFO[0016] Approved InstallPlan install-44bh9 for the Subscription: nginx-operator-v0-0-1-sub 
INFO[0016] Waiting for ClusterServiceVersion "default/nginx-operator.v0.0.1" to reach 'Succeeded' phase 
INFO[0016]   Waiting for ClusterServiceVersion "default/nginx-operator.v0.0.1" to appear 
INFO[0023]   Found ClusterServiceVersion "default/nginx-operator.v0.0.1" phase: Pending 
INFO[0026]   Found ClusterServiceVersion "default/nginx-operator.v0.0.1" phase: Installing 
INFO[0046]   Found ClusterServiceVersion "default/nginx-operator.v0.0.1" phase: Succeeded
INFO[0046] OLM has successfully installed "nginx-operator.v0.0.1"
```

#### Giải thích chi tiết

* `operator-sdk run bundle <bundle-image>`:

  * tạo các resource OLM cần thiết
  * feed bundle vào OLM
  * chờ CSV của Operator chuyển sang `Succeeded`

Các resource được tạo tự động gồm:

* `CatalogSource`: nguồn danh mục Operator
* `Subscription`: khai báo “tôi muốn cài package này, channel này”
* `ClusterServiceVersion (CSV)`: mô tả version và metadata của Operator
* `OperatorGroup`: xác định phạm vi namespace mà Operator quản lý
* `InstallPlan`: kế hoạch cài đặt do OLM tạo ra

### Ghi nhớ quan trọng

Nếu `ClusterServiceVersion` fail khi cài:

* kiểm tra bạn đã cài `kube-prometheus-stack` ở Chương 6 chưa
* nếu bundle tham chiếu tới Prometheus resources mà cluster chưa có CRD tương ứng, cài đặt có thể fail. 

---

#### 2. Gỡ Operator đã cài bằng OLM

```bash
$ operator-sdk cleanup nginx-operator

INFO[0000] subscription "nginx-operator-v0-0-1-sub" deleted 
INFO[0000] customresourcedefinition "nginxoperators.operator.example.com" deleted 
INFO[0000] clusterserviceversion "nginx-operator.v0.0.1" deleted 
INFO[0000] catalogsource "nginx-operator-catalog" deleted 
INFO[0000] operatorgroup "operator-sdk-og" deleted      
INFO[0000] Operator "nginx-operator" uninstalled
```

#### Giải thích

* `cleanup <packageName>`: gỡ các resource OLM liên quan tới package Operator
* `nginx-operator` là tên project/package trong file `PROJECT`. 

### Ghi nhớ

* `operator-sdk cleanup` là cleanup cho **Operator đã cài bằng OLM**
* khác với `olm uninstall` là gỡ **chính OLM**

---

## Working with OperatorHub

### Kiến thức lý thuyết cần nhớ

OperatorHub là catalog trung tâm cho Operators do cộng đồng publish. Mục tiêu của nó là tạo một nơi tập trung để:

* khám phá Operators
* tìm metadata về Operator
* cài Operator
* phân phối Operator ra cộng đồng. 

Sách nhấn mạnh:

* OperatorHub được ra mắt năm 2019
* là động lực lớn cho việc phổ biến Kubernetes Operators
* mô hình quản trị mở, dựa trên GitHub repo public và maintainer cộng đồng. 

### Ghi nhớ

* OperatorHub = nơi **discover** và **promote** Operators
* OLM = nơi **install/manage** Operators trong cluster
* CSV là điểm nối quan trọng giữa OLM và OperatorHub

---

## Installing Operators from OperatorHub

### Kiến thức lý thuyết cần nhớ

Cài Operator từ OperatorHub là workflow “người dùng cuối” của hệ sinh thái:

1. tìm Operator trên website
2. mở trang chi tiết
3. nhấn **Install**
4. copy lệnh cài
5. OLM thực hiện phần còn lại. 

### Ví dụ trong sách: Grafana Operator

#### 1. Cài bằng YAML install URL

```bash
$ kubectl create -f https://operatorhub.io/install/grafana-
operator.yaml

namespace/my-grafana-operator created
operatorgroup.operators.coreos.com/operatorgroup created
subscription.operators.coreos.com/my-grafana-operator created
```

#### Giải thích chi tiết

* `kubectl create -f <url>`: tải manifest YAML từ URL rồi tạo resource trên cluster
* manifest này tạo:

  * namespace cho operator
  * `OperatorGroup`
  * `Subscription`. 

---

#### 2. Kiểm tra resource sau khi cài

```bash
$ kubectl get all -n my-grafana-operator

NAME                                                      READY   
STATUS   pod/grafana-operator-controller-manager-b95954bdd-
sqwzr   2/2     Running   

NAME                                                          
TYPE        service/grafana-operator-controller-manager-
metrics-service   ClusterIP  

NAME                                                  READY   
UP-TO-DATE   deployment.apps/grafana-operator-controller-
manager   1/1     1            1           

NAME                                                            
DESIRED   replicaset.apps/grafana-operator-controller-manager-
b95954bdd   1         1
```

#### Giải thích

* xác nhận Operator đã được OLM cài thành công
* Pod đang `Running`
* deployment sẵn sàng
* metrics service đã tồn tại. 

### Kiến thức cần nhớ về OperatorGroup và Subscription

* `OperatorGroup`: biểu diễn phạm vi cài đặt / namespace scope cho Operator
* `Subscription`: biểu diễn ý định của người dùng muốn cài một Operator từ catalog

OLM đọc các object này để tạo:

* Deployment
* Service
* và các resource khác cần cho Operator. 

### Ghi nhớ

Sau khi cài Operator:

* thường bạn còn phải tạo **CR cấu hình riêng** của Operator đó
* nhiều CRD khác như `OperatorGroup`, `Subscription` là do OLM tự quản lý, người dùng thường không phải chỉnh tay. 

---

## Submitting your own Operator to OperatorHub

### Kiến thức lý thuyết cần nhớ

Public Operator không bắt buộc, nhưng có lợi:

* tăng độ nhận diện cho ứng dụng/công ty
* thể hiện mức đầu tư vào vận hành chuẩn hóa
* giúp người dùng cài và sử dụng dễ hơn. 

Workflow submit lên OperatorHub gồm 3 phần chính:

1. tạo bundle
2. build bundle image
3. submit metadata/bundle vào community operators repository. 

---

### Các bước submit theo sách

#### 1. Fork community-operators repo

Repo:
`https://github.com/k8s-operatorhub/community-operators`

#### 2. Tạo thư mục cho Operator

Ví dụ:

```text
operators/nginx-operator
```

#### 3. Tạo `ci.yaml`

```yaml
reviewers:
  - myGithubUsername
  - yourTeammateUsername
```

#### Giải thích

* `reviewers`: danh sách GitHub usernames của reviewer/maintainer phụ trách Operator. 

---

#### 4. Tạo thư mục version

Ví dụ:

```text
operators/nginx-operator/0.0.1
```

#### 5. Copy bundle vào thư mục version

Copy:

* thư mục `bundle/`
* file `bundle.Dockerfile`. 

#### 6. Commit + push lên fork của bạn

#### 7. Tạo Pull Request vào upstream repo

#### 8. Đọc PR template và hoàn tất pre-submit checks

Sách nhấn mạnh các check chính:

* xem contribution guidelines
* test Operator trong local cluster
* kiểm tra metadata hợp chuẩn OperatorHub
* đảm bảo mô tả và versioning rõ ràng cho người dùng. 

### Ghi nhớ

* Nếu PR pass automation checks, nó sẽ được merge
* sau đó Operator sẽ xuất hiện trên OperatorHub một thời gian ngắn sau. 

---

## Troubleshooting

### Ý chính

Phần này không đưa thêm nhiều lệnh debug mới mà chủ yếu chỉ đường tới nơi nên tìm trợ giúp khi vướng OLM hoặc OperatorHub. 

---

## OLM support

### Nguồn hỗ trợ

* Slack: `#operator-sdk-dev` trên `slack.k8s.io`
* GitHub:
  `https://github.com/operator-framework/operator-lifecycle-manager`
* Docs:
  `https://sdk.operatorframework.io/docs/olm-integration/`. 

### Ghi nhớ

Khi lỗi ở các chỗ như:

* `olm install`
* `olm status`
* `run bundle`
* dependency/resources do OLM tạo
  thì nên tra OLM docs + GitHub issues trước.

---

## OperatorHub support

### Nguồn hỗ trợ

* Community catalog repo:
  `https://github.com/k8s-operatorhub/community-operators`
* Frontend site code:
  `https://github.com/k8s-operatorhub/operatorhub.io`
* Submission docs:
  `https://k8s-operatorhub.github.io/community-operators/`. 

### Preview tool

OperatorHub có tool preview submission:
`https://operatorhub.io/preview`

### Ý nghĩa

* preview cách Operator của bạn sẽ hiện trên OperatorHub
* kiểm tra metadata có hợp lệ cú pháp không
* xác nhận CSV/CRD metadata đang hiển thị đúng như mong muốn. 

### Ghi nhớ

Preview là bước thủ công rất hữu ích vì:

* CSV/CRD khá dài và dễ rối
* nhìn trước giao diện hiển thị giúp bắt lỗi metadata sớm. 

---

## Summary

### Kiến thức tổng kết cần nhớ

Chương này hoàn tất workflow chính để biến một Operator từ:

* code đang phát triển
* sang sản phẩm có thể cài và phân phối công khai. 

Trình tự lớn của chương:

1. cài OLM
2. kiểm tra OLM hoạt động đúng
3. tạo bundle cho Operator
4. hiểu cấu trúc bundle và CSV
5. build bundle image
6. push bundle image
7. chạy Operator qua OLM
8. cài Operator từ OperatorHub
9. submit Operator của chính bạn lên OperatorHub. 

### Những ý quan trọng nhất cần nhớ

* **OLM** là cách chuẩn để install/manage Operator trong cluster
* **Bundle** là định dạng OLM hiểu được
* **CSV** là file metadata trung tâm của bundle
* **Bundle image** phải được push lên registry
* **OperatorHub** là catalog công khai để phân phối và khám phá Operators
* Sau khi release, vòng đời Operator **chưa kết thúc**; bước tiếp theo là bảo trì, versioning, deprecation, release management ở Chương 8. 