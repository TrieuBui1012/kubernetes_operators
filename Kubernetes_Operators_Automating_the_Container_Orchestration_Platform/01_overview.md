# Tổng quan về Operator

## 1. Vấn đề của Kubernetes: Ứng dụng Không trạng thái (Stateless) vs. Có trạng thái (Stateful)
- **Quản lý ứng dụng Stateless rất dễ:** Kubernetes tự động hóa vòng đời của các ứng dụng không lưu trạng thái (ví dụ như một máy chủ web tĩnh) một cách xuất sắc. Vì các bản sao (replica) ứng dụng này hoàn toàn giống nhau, nếu một pod bị lỗi, Kubernetes chỉ cần thay thế nó bằng một pod mới mà không cần bất kỳ sự chuẩn bị đặc biệt nào,.
- **Ứng dụng Stateful lại rất khó:** Hầu hết các ứng dụng thực tế (như cơ sở dữ liệu) đều cần lưu trạng thái, đòi hỏi những quy trình khởi động, thiết lập cụm và lưu trữ dữ liệu phức tạp. Kubernetes được thiết kế để giữ tính linh hoạt và tổng quát, do đó nó không thể tự hiểu được các thiết lập cấu hình đặc thù cho từng phần mềm cụ thể (ví dụ như cách quản lý một cụm PostgreSQL).

## 2. Operator hoạt động như một kỹ sư SRE tự động (Software SREs)
- **Triết lý SRE:** SRE (Site Reliability Engineering) là tập hợp các nguyên tắc tập trung vào việc tự động hóa quá trình quản trị hệ thống bằng phần mềm, giúp các nhóm giảm bớt công việc bảo trì lặp đi lặp lại để tập trung vào phát triển sản phẩm.
- **Đưa kỹ năng chuyên gia vào phần mềm:** Một Operator đóng vai trò như một SRE phần mềm cho một ứng dụng cụ thể. Nó chuyển các kỹ năng của một chuyên gia quản trị vào code, cho phép tự động hóa các tác vụ từ cài đặt hệ thống, theo dõi ứng dụng đang chạy, sao lưu dữ liệu, khôi phục sau sự cố, cho đến tự động nâng cấp phiên bản.

## 3. Cơ chế hoạt động của Operator
- Operator hoạt động bằng cách mở rộng mặt phẳng điều khiển (control plane) và API của chính Kubernetes. Quá trình tạo và vận hành Operator dựa trên 2 thành phần:
    - **Tài nguyên Tùy chỉnh (Kubernetes CRs):** Thông qua Custom Resource Definition (CRD), Operator định nghĩa một điểm cuối (endpoint) API mới để người dùng có thể thao tác bằng các công cụ chuẩn như `kubectl`,. CR lưu trữ trạng thái cấu hình mong muốn của ứng dụng.
    - **Bộ điều khiển Tùy chỉnh (Custom Controllers):** Operator về cơ bản là một bộ điều khiển tùy chỉnh chạy trong một vòng lặp liên tục, chuyên theo dõi một loại CR cụ thể. Nó sẽ thực thi các hành động mang tính đặc thù của ứng dụng để đảm bảo trạng thái thực tế của hệ thống luôn khớp với những gì được khai báo trong CR.

## 4. Ví dụ minh họa: etcd Operator
- etcd là một cơ sở dữ liệu phân tán cần sự quản trị từ chuyên gia để xử lý việc thêm node, nâng cấp và sao lưu.
- Với **etcd Operator**, nếu một pod của cụm etcd bị xóa ngẫu nhiên, Operator sẽ tự động phát hiện và không chỉ tạo lại pod mới (như Kubernetes cơ bản vẫn làm) mà còn thiết lập kết nối, tái tạo cấu hình để pod mới này gia nhập vào cụm hiện tại một cách an toàn,.

## 5. Đối tượng sử dụng và sự đón nhận
- **Quản trị viên cụm và người dùng:** Operator giúp họ dễ dàng sử dụng các phần mềm nền tảng phức tạp mà không cần phải là chuyên gia về phần mềm đó (chẳng hạn như dễ dàng thiết lập một cụm cơ sở dữ liệu).
- **Lập trình viên ứng dụng và Kỹ sư hạ tầng:** Xây dựng Operator để đơn giản hóa quá trình triển khai sản phẩm đến tay khách hàng và kiểm soát hệ thống hiệu quả hơn,.
- **Mức độ phổ biến:** Pattern (mẫu thiết kế) Operator đã được đón nhận rộng rãi với vô số Operator dành cho PostgreSQL, MongoDB, Redis hay lưu trữ Ceph (Rook). Các nền tảng như Red Hat OpenShift thậm chí sử dụng Operator như một phương tiện cốt lõi để xây dựng và cập nhật chính bản thân nền tảng của họ.