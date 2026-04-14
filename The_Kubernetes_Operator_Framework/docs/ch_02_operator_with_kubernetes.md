# Operators interact with Kubernetes

## 1. Operator trước hết là một thành phần làm việc với tài nguyên Kubernetes

### Pods, ReplicaSets, Deployments

* **Pod** là đơn vị chạy ứng dụng cơ bản nhất.
* **ReplicaSet** đảm bảo số lượng Pod replica mong muốn.
* **Deployment** quan trọng hơn trong thực tế vì ngoài replica còn hỗ trợ rollout, revision và rollback.
* Với nhiều Operator, **Deployment là resource chính để quản lý Operand**, còn bản thân Operator cũng thường được chạy bằng Deployment. 

### CRDs là trung tâm của Operator

* Hầu hết Operator dựa vào **CustomResourceDefinition (CRD)**.
* CRD cho phép mở rộng Kubernetes API bằng resource mới.
* Người dùng thường không thao tác trực tiếp với Deployment của Operator, mà thao tác qua **custom resource** do CRD định nghĩa.
* Nhờ đó, Operator tạo ra một lớp giao diện rõ ràng hơn, che bớt độ phức tạp nội bộ của Kubernetes. 

### RBAC là bắt buộc

* Operator cần **RBAC** để quản lý Operand và cả CRD của chính nó.
* Chuỗi quyền cơ bản là:

  * **Role / ClusterRole**: định nghĩa quyền
  * **ServiceAccount**: danh tính của Pod
  * **RoleBinding / ClusterRoleBinding**: gắn quyền vào danh tính
* Không có chuyện Operator tự nhiên có quyền với CRD của mình; nó vẫn phải được cấp quyền như mọi API object khác. 

### Namespace và scope

* Operator có thể là:

  * **namespace-scoped**: chỉ quản lý trong một namespace
  * **cluster-scoped**: quản lý nhiều namespace
* Chọn scope phụ thuộc vào loại resource mà Operator cần quản lý.
* Namespace-scoped dễ debug và cô lập lỗi hơn; cluster-scoped phù hợp khi Operand hoặc yêu cầu trải rộng nhiều namespace. 

## 2. Operator không chỉ tương tác với cluster, mà còn tương tác với con người

### Cluster administrators

* Là nhóm có quyền cao nhất và hiểu cluster nhiều nhất.
* Họ cần quyền lực và độ linh hoạt cao, nhưng cũng dễ gây rủi ro nếu Operator phơi bày quá nhiều cấu hình nguy hiểm.
* Vì vậy, khi thiết kế cho admin, cần cân bằng giữa kiểm soát mạnh và an toàn, dễ hỗ trợ
* Trong tình huống sự cố nghiêm trọng, admin có thể cần quyền truy cập trực tiếp vào Operator hoặc Operand để xử lý khẩn cấp. 

### Cluster users

* Là người dùng nội bộ có tương tác với cluster nhưng không có toàn quyền như admin.
* Ví dụ: developer yêu cầu provision môi trường, cấp tài nguyên tạm thời.
* Nhóm này thường nên được cấp ít quyền hơn, và đôi khi chỉ nên đi qua một ứng dụng nội bộ thay vì chạm trực tiếp vào Operator.
* Điều này giúp giảm rủi ro mà vẫn giữ được trải nghiệm dùng tốt. 

### End users / customers

* Phần lớn người dùng cuối không tương tác trực tiếp với Operator.
* Họ chỉ dùng frontend hoặc ứng dụng bên ngoài, còn Operator nằm ở backend.
* Trong bối cảnh này, điều quan trọng là Operator phải có API tốt để giao tiếp với chương trình khác, chứ không nhất thiết tối ưu cho người dùng con người thao tác trực tiếp. 

### Maintainers

* Maintainer là người sửa lỗi, thêm tính năng, và giữ cho Operator sống lâu dài.
* Nếu dự án open source, cần có trusted owners review thay đổi code.
* Với Operator, maintainer nên hiểu rõ các khái niệm lõi của Kubernetes vì Operator phụ thuộc rất mạnh vào chúng.
* Đội ngũ maintainers tốt là yếu tố quan trọng để Operator tồn tại lâu dài. 

## 3. Tính năng tốt là tính năng giải quyết vấn đề thật

### Tránh dư thừa

* Không nên viết lại một Operator đã tồn tại nếu không có lý do đủ mạnh.
* Cũng không nên “reinvent the wheel” ở cấp độ khái niệm Kubernetes, tức là tự xây lại thứ Kubernetes đã giải quyết tốt. 

### Tránh giải quyết vấn đề giả định

* Nhiều ý tưởng nghe rất hợp lý nhưng thực tế không có bằng chứng người dùng cần.
* Mỗi tính năng mới đều có chi phí:

  * chi phí xây dựng
  * chi phí bảo trì
  * chi phí hỗ trợ về sau
* Vì vậy, nên xác thực đề xuất tính năng bằng nhu cầu thật thay vì chỉ bằng cảm giác. 

### Câu hỏi quan trọng

* Khi nghĩ về một tính năng, luôn nên tự hỏi:

  * **“Tính năng này hữu ích ở điểm nào?”**
* Cách suy nghĩ này giúp tránh thêm các thay đổi không cần thiết, vốn sẽ biến thành gánh nặng bảo trì trong tương lai. 

## 4. Operator phải được thiết kế để thay đổi tốt theo thời gian

Chương này xem thay đổi là điều chắc chắn sẽ xảy ra: bug fix, refactor, tính năng mới, tính năng cũ bị bỏ đi. Với Operator, điều đó còn rõ hơn vì nó phụ thuộc vào chính Kubernetes và hệ sinh thái xung quanh. Sách gói gọn phần này trong 3 nguyên tắc: **start small, iterate effectively, deprecate gracefully**. 

### Start small

* Đừng cố nhồi quá nhiều tính năng ngay từ đầu.
* Một Operator quá nhiều option sẽ làm mất lợi ích của abstraction và automation.
* Phiên bản đầu nên tập trung vào **mục tiêu cốt lõi**, tạo nền tảng chắc trước rồi mới mở rộng. 

### Iterate effectively

* Thêm tính năng mới thường dễ hơn bỏ tính năng cũ.
* Vì vậy nên:

  * chủ động lấy feedback người dùng
  * theo dõi usage
  * quan sát thay đổi từ cộng đồng Kubernetes
* Mỗi lần mở rộng phải kiểm tra xem tính năng mới có còn phù hợp với **user base** và **mục tiêu thiết kế ban đầu** hay không. 

### Deprecate gracefully

* Deprecation là một trong những thay đổi gây khó chịu nhất cho người dùng.
* Nếu buộc phải bỏ tính năng hoặc đổi API, cần:

  * thông báo sớm
  * cho thời gian chuyển đổi
  * giữ cách làm nhất quán với tinh thần deprecation của Kubernetes
* Mục tiêu là thay đổi có trách nhiệm, ít gây sốc, dễ dự đoán. 