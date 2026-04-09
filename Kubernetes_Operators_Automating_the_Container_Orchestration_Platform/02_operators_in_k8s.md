# Các thành phần của Operator trên Kubernetes

## 1. Cơ chế mở rộng tiêu chuẩn: Tài nguyên ReplicaSet (Standard Scaling: The ReplicaSet Resource)
- Trong Kubernetes, các tài nguyên tiêu chuẩn như **ReplicaSet** hoạt động như một bộ sưu tập các đối tượng pod để tạo thành một danh sách các bản sao ứng dụng đang chạy.
- Một thành phần tiêu chuẩn được gọi là **ReplicaSet controller** sẽ liên tục giám sát, so sánh số lượng pod thực tế với trạng thái mong muốn và tự động khởi tạo hoặc xóa pod để đảm bảo sự đồng nhất.
- Tuy nhiên, các bộ điều khiển tiêu chuẩn này mang tính chất tổng quát và không quan tâm đến đặc thù của ứng dụng (application agnostic). Chúng không thể hiểu được các quy trình khởi động, tắt máy hay cấu hình phức tạp của từng phần mềm cụ thể.
- Để giải quyết vấn đề này, **Operator** ra đời, đóng vai trò là sự kết hợp giữa các tài nguyên tùy chỉnh (CR) và một bộ điều khiển tùy chỉnh (custom controller) thấu hiểu mọi chi tiết về ứng dụng cụ thể mà nó quản lý.

## 2. Custom Resources - CRs
- CR là cơ chế mở rộng API của Kubernetes, cung cấp các điểm cuối (endpoint) để đọc và ghi dữ liệu có cấu trúc. Người dùng có thể tương tác với CR thông qua các công cụ chuẩn như `kubectl`.
- **Phân biệt CR và ConfigMap:** Trong khi ConfigMap chủ yếu dùng để truyền tệp cấu hình trực tiếp vào cho một chương trình chạy bên trong pod, thì CR lại là một tập hợp các đối tượng API được giám sát bởi các bộ điều khiển tùy chỉnh, từ đó tự động tạo ra hoặc thay đổi các đối tượng khác trong cụm.

## 3. Custom Controllers
- Bản thân một CR chỉ chứa dữ liệu lưu trong cơ sở dữ liệu API chứ không có khả năng tự thực thi. 
- Vì vậy, mỗi Operator đều sở hữu một hoặc nhiều **custom controllers** (bộ điều khiển tùy chỉnh). Các bộ điều khiển này chạy vòng lặp liên tục để giám sát sự thay đổi trên CR và tự động thực thi các hành động bảo trì, quản lý ứng dụng trên thực tế sao cho khớp với cấu hình mà người dùng đã khai báo.

## 4. Operator Scopes
- **Namespace Scope:** Đây là phạm vi được khuyến nghị trong hầu hết các trường hợp. Việc giới hạn Operator hoạt động trong một namespace duy nhất mang lại tính linh hoạt cao, cho phép các nhóm chạy hoặc nâng cấp các phiên bản Operator/ứng dụng một cách độc lập mà không ảnh hưởng tới toàn bộ hệ thống.
- **Cluster-Scoped Operators:** Một số Operator đặc thù cần quản lý tài nguyên trên toàn bộ cụm, ví dụ như hệ thống service mesh (Istio) hay cấp phát chứng chỉ TLS (cert-manager).

## 5. Authorization & RBAC
Vì Operator mở rộng chính bản thân Kubernetes, nó cần được cấp quyền thông qua hệ thống **Role-Based Access Control (RBAC)** của nền tảng. Các thành phần ủy quyền bao gồm:
- **Service Accounts:** Thay vì dùng tài khoản của người dùng thực, Kubernetes sử dụng Service Account để cung cấp danh tính và ủy quyền cho các chương trình phần mềm, điển hình là Operator.
- **Roles:** Theo mặc định, Kubernetes từ chối mọi quyền. Quản trị viên phải tạo ra các Role để định nghĩa rõ ràng những quyền hạn (ví dụ: thao tác đọc, tạo, cập nhật, xóa) đối với các tài nguyên cụ thể.
- **RoleBindings:** Là đối tượng giúp gán một Role cụ thể cho một Service Account giới hạn trong một namespace.
- **ClusterRoles và ClusterRoleBindings:** Đây là các khái niệm tương đương của Role và RoleBinding nhưng có mức độ áp dụng trên toàn bộ cụm thay vì chỉ trong một namespace, thường được dùng cho các Cluster-Scoped Operator.