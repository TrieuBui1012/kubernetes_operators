# Chương 1 – Tóm tắt các ý chính quan trọng

## 1. Vì sao cần Operator Framework?

Kubernetes rất mạnh, nhưng việc vận hành ứng dụng trên đó nhanh chóng trở nên phức tạp vì một ứng dụng thường gồm nhiều thành phần như Pod, Service, Volume, database, cấu hình, credentials và các dependency khác. Khi có thay đổi cấu hình, rollout phiên bản mới hoặc lỗi bất ngờ, đội ngũ vận hành phải tốn nhiều công sức thủ công để giữ hệ thống ổn định. Operator Framework ra đời để giải quyết chính bài toán tự động hóa các tác vụ vận hành này. 

## 2. Operator thực chất là gì?

Operator về bản chất là một **controller** của Kubernetes, nhưng được viết chuyên biệt cho một workload hoặc thành phần cụ thể. Nó liên tục quan sát trạng thái hiện tại của cluster, so sánh với trạng thái mong muốn, rồi tự động **reconcile** để đưa hệ thống về đúng trạng thái cần có. 

## 3. Các khái niệm cốt lõi

### Operand

Là ứng dụng hoặc workload mà Operator quản lý. Operand có thể là ứng dụng của bạn, thành phần hỗ trợ như backup system, hoặc thậm chí là thành phần lõi của cluster như etcd. 

### Custom Resource và CRD

Operator cung cấp giao diện cho người dùng thông qua **Custom Resource (CR)**. CR này được định nghĩa bởi **CustomResourceDefinition (CRD)**, tức cơ chế cho phép mở rộng API của Kubernetes bằng các loại resource mới. Nhờ đó, người dùng có thể thao tác với Operator gần giống như với các resource native của Kubernetes. 

## 4. Ba trụ cột của Operator Framework

### Operator SDK

Là bộ công cụ để phát triển Operator. Nó hỗ trợ scaffold project, tạo API, sinh mã, cung cấp pattern phát triển và CLI `operator-sdk` để khởi tạo cũng như quản lý vòng đời phát triển Operator. 

### OLM (Operator Lifecycle Manager)

Là thành phần giúp cài đặt, nâng cấp và quản lý Operator trong cluster. OLM còn theo dõi các Operator đã cài, hỗ trợ dependency management và giúp phát hiện xung đột giữa các Operator để tăng độ ổn định cho cluster. 

### OperatorHub

Là catalog mở để phân phối và chia sẻ Operator cho cộng đồng. Operator được public thông qua bundle/manifest chuẩn, đặc biệt dựa vào **CSV (Cluster Service Version)** để OLM và OperatorHub hiểu metadata, CRD, dependency và cách cài đặt của Operator. 

## 5. Capability Model

Operator Framework dùng **Capability Model** để phân loại mức độ trưởng thành và chức năng của Operator. Có 5 cấp:

### Level I – Basic Install

Operator chỉ cần cài Operand và báo trạng thái cơ bản. 

### Level II – Seamless Upgrades

Bổ sung khả năng nâng cấp Operand và cả chính Operator một cách mượt mà, có tính đến backward compatibility và rollback. 

### Level III – Full Lifecycle

Operator bắt đầu quản lý vòng đời sâu hơn của Operand như backup/restore, failover/failback, scaling, quản lý cluster member, workflow phức tạp hơn. 

### Level IV – Deep Insights

Operator cung cấp metrics, events, alerts và các insight đo lường được về sức khỏe, hiệu năng của chính nó và của Operand. 

### Level V – Auto Pilot

Mức cao nhất: Operator có thể tự động scale, tự chữa lỗi, tự tuning và phát hiện bất thường gần như theo kiểu “tự lái”. 

## 6. Giá trị thực sự của Operator

Điểm quan trọng nhất của Operator là nó **đóng gói tri thức vận hành** thành phần mềm. Thay vì quản trị viên phải tự thao tác với từng Deployment, Pod, Service và config, họ chỉ cần tương tác với một CRD ở mức cao hơn. Operator sẽ chịu trách nhiệm triển khai, cập nhật, theo dõi trạng thái, xử lý lỗi và giữ ứng dụng vận hành đúng như mong muốn. 