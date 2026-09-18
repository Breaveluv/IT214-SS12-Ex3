# BÁO CÁO PHÂN TÍCH: NGUYÊN NHÂN CIRCUIT BREAKER KHÔNG MỞ KHI LỖI 100%

## 1. Tóm tắt sự cố
- **Hệ thống:** Payment-Service của StoreX tích hợp với API của Ngân hàng đối tác.
- **Cấu hình ban đầu:**
  - `slidingWindowSize = 20` (dựa trên 20 request gần nhất - `COUNT_BASED`)
  - `failureRateThreshold = 50%`
  - `waitDurationInOpenState = 30s`
  - `permittedNumberOfCallsInHalfOpenState = 3`
  - *Không khai báo tham số `minimumNumberOfCalls`.*
- **Hiện tượng:** Vào ban đêm (khung giờ thấp điểm), có đúng 5 khách hàng thực hiện thanh toán và cả 5 giao dịch đều thất bại (tỷ lệ lỗi đạt **100%**, vượt xa ngưỡng 50%). Tuy nhiên, Circuit Breaker **vẫn ở trạng thái `CLOSED`** và không hề ngắt mạch.

---

## 2. Phân tích nguyên nhân kỹ thuật cốt lõi (Root Cause)

### 2.1. Vai trò của `minimumNumberOfCalls` trong Resilience4j
Trong Resilience4j, Circuit Breaker được thiết kế để tránh "báo động giả" (false positive) do biến động tức thời khi số lượng mẫu còn quá ít. Vì vậy, nó sử dụng tham số:
> **`minimumNumberOfCalls`**: Số lượng cuộc gọi tối thiểu phải được ghi nhận trong Sliding Window trước khi Circuit Breaker bắt đầu tính toán tỷ lệ lỗi (`failureRate`) để quyết định đóng/ngắt mạch.

### 2.2. Giá trị mặc định khi không cấu hình
Khi người phát triển chỉ cấu hình `slidingWindowSize = 20` mà **không khai báo tường minh `minimumNumberOfCalls`**, Resilience4j sẽ tự động gán:
$$\text{minimumNumberOfCalls} = \text{slidingWindowSize} = 20$$

### 2.3. Đối chiếu với kịch bản thực tế
- Số request thực tế gửi đến trong đêm: $N_{thuc\_te} = 5$
- Ngưỡng tối thiểu để tính toán tỷ lệ lỗi: $N_{min} = 20$
- Điều kiện kiểm tra:
  $$N_{thuc\_te} (5) < N_{min} (20)$$

**Kết luận:** Mặc dù tỷ lệ lỗi cục bộ là $5/5 = 100\%$, nhưng vì số lượng mẫu chưa đạt ngưỡng tối thiểu ($5 < 20$), Resilience4j xác định dữ liệu chưa đủ độ tin cậy thống kê (statistical significance). Do đó, Circuit Breaker **bỏ qua bước tính toán tỷ lệ lỗi và tiếp tục duy trì trạng thái `CLOSED`**.

---

## 3. Đánh giá hậu quả nghiệp vụ

1. **Vi phạm cam kết SLA với đối tác:**
   - Ngân hàng đối tác yêu cầu: *"Nếu hệ thống của chúng tôi lỗi quá 50%, các bạn phải ngắt kết nối trong vòng 30 giây để chúng tôi sửa chữa"*.
   - Việc cầu dao không mở khiến StoreX vi phạm trực tiếp thỏa thuận này.
2. **Gây hiệu ứng "dội bom" (Traffic Bombing):**
   - Phía ngân hàng đang gặp sự cố và cần thời gian phục hồi, nhưng StoreX vẫn liên tục gửi request sang, làm tình trạng quá tải/sập hệ thống của đối tác nghiêm trọng hơn.
3. **Lãng phí tài nguyên và làm suy giảm trải nghiệm người dùng:**
   - Các luồng (thread) xử lý của Payment-Service bị giữ lại để chờ timeout từ ngân hàng, dẫn đến nguy cơ cạn kiệt connection pool cục bộ.

---

## 4. Giải pháp khắc phục

Để Circuit Breaker phản ứng nhanh nhạy trong các khung giờ thấp điểm (low-traffic), cần cấu hình tường minh `minimumNumberOfCalls` với các ràng buộc:

1. **Ràng buộc nghiệp vụ & kỹ thuật:**
   $$\text{minimumNumberOfCalls} \ge \text{permittedNumberOfCallsInHalfOpenState} = 3$$
2. **Giá trị tối ưu:** Chọn $\mathbf{minimumNumberOfCalls = 3}$.

### Diễn biến khi đã áp dụng khắc phục:
- Khi có 5 khách hàng gửi request trong đêm:
  - Request 1, 2 bị lỗi: Tổng 2 cuộc gọi, chưa đủ $N_{min} = 3$.
  - **Request 3 bị lỗi:** Tổng 3 cuộc gọi, đã đủ $N_{min} = 3$. Tỷ lệ lỗi lúc này là $3/3 = 100\% \ge 50\%$.
  - $\rightarrow$ **Circuit Breaker lập tức chuyển sang trạng thái `OPEN` ngay tại request thứ 3.**
  - **Request 4 và 5:** Bị chặn lại ngay lập tức tại Payment-Service (Fail-fast / Fallback), hoàn toàn không gửi sang ngân hàng đối tác trong vòng 30 giây, đáp ứng 100% yêu cầu nghiệp vụ.
