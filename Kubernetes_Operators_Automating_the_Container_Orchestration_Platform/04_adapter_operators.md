# Adapter Operator

## Tổng quan 

- Không viết logic reconcile bằng Go từ đầu, mà dùng công cụ sẵn (helm, ansible) có rồi để Operator SDK tạo phần vỏ controller, CRD, RBAC, bundle OLM.

### Khi nào nên sử dụng

- Thứ nhất: không phải lúc nào cũng nên viết Operator bằng Go.
Nếu team đã có Helm chart hoặc Ansible role chạy tốt, có thể nâng cấp nó thành Operator rất nhanh.

- Thứ hai: người dùng cuối không còn chạy:

```
helm install ...
```

hay:

```
ansible-playbook ...
```

mà họ chỉ cần tạo một Custom Resource như:

```
apiVersion: app.example.com/v1alpha1
kind: VisitorsApp
metadata:
  name: demo
spec:
  replicas: 2
  title: "Visitors Dashboard"
```

Operator sẽ đọc CR này rồi dùng Helm/Ansible ở bên trong để tạo Deployment, Service, Secret, v.v.

- Thứ ba: lợi ích lớn không chỉ là deploy được, mà là:
  - có API Kubernetes chuẩn,
  - có thể đóng gói với OLM/bundle,
  - có vòng đời quản lý tốt hơn so với chạy Helm/Ansible rời rạc.

- Thứ tư: đây là cách rất hợp để bắt đầu:
  - đã có chart → dùng Helm Operator
  - đã có automation bằng Ansible → dùng Ansible Operator
  - cần logic reconcile phức tạp, nhiều nhánh, phản ứng động → sang Go Operator.

# Helm-based operator:

## Create a new helm chart

```
mkdir visitors-helm-operator
cd visitors-helm-operator

operator-sdk init --plugins helm --domain trieubui1012.com

operator-sdk create api \
  --group example \
  --version v1 \
  --kind VisitorsApp
```

## Use an existing chart

```
operator-sdk init --plugins helm --domain mycompany.com

# local helm chart
operator-sdk create api \
  --helm-chart ./visitors-helm.tgz \
  --group example \
  --version v1 \
  --kind VisitorsApp

# remote helm chart
operator-sdk create api \
  --group visitors \
  --version v1 \
  --kind VisitorsApp \
  --helm-chart nginx \ 
  --helm-chart-repo https://charts.bitnami.com/bitnami \
  --helm-chart-version 15.9.0
```

--helm-chart: the name of helm chart (when using remote) or the .tgz file or directory (when using local)
--helm-chart-repo: the helm chart repo URL
--helm-chart-version: the helm chart version
