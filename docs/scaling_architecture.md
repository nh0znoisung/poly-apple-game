# Kiến Trúc Mở Rộng Poly Apple (Quy Mô 1000 CCU & Hệ Thống Xếp Hạng)

Để đưa Poly Apple từ một dự án MVP lên quy mô thực tế phục vụ **1000 người chơi cùng lúc (CCU)** với các tính năng như Đăng nhập, Lịch sử, và Rank, bạn **không cần phải đập đi xây lại toàn bộ**, nhưng **BẮT BUỘC** phải nâng cấp một số phần trong Tech Stack. 

Dưới đây là kế hoạch chuyển đổi và phân tích Tech Stack:

---

## 1. Hệ Quản Trị Cơ Sở Dữ Liệu (Bắt buộc đổi)

> [!CAUTION]
> **Hiện tại:** Đang dùng file JSON (`fs.writeFileSync`). Với 1000 CCU, việc đọc/ghi liên tục vào ổ cứng sẽ gây hiện tượng nghẽn cổ chai (I/O Blocking), làm giật lag server và có nguy cơ mất dữ liệu nếu crash.

**Stack Đề Xuất:**
- **MongoDB (Database Chính):** Rất may mắn là hiện tại cấu trúc `players`, `rooms`, `sessions` tôi đã thiết kế theo chuẩn Document của MongoDB. Việc chuyển đổi gần như chỉ là thay hàm đọc/ghi file thành lệnh gọi MongoDB. Dùng để lưu trữ: Profile người chơi, Lịch sử trận đấu, Lịch sử biểu thức.
- **Redis (In-Memory Cache):** Dùng để quản lý các **Phòng đang chơi (Rooms)** và **Hàng đợi tìm trận (Matchmaking Queue)**. Tốc độ đọc/ghi của Redis nằm trên RAM nên độ trễ gần như bằng không (microseconds), hoàn hảo cho Game Real-time.

---

## 2. Backend & Real-time (Giữ nguyên nhưng cần Tối ưu)

> [!NOTE]
> **Node.js + Socket.io** sinh ra là để xử lý I/O bất đồng bộ. Một con VPS cấu hình vừa phải (2 Core, 4GB RAM) dư sức gánh 5,000 - 10,000 connection Socket.io cùng lúc. 1000 CCU là con số rất nhỏ với Node.js.

**Điều chỉnh cần làm:**
- **Scale ngang (Horizontal Scaling):** Nếu vượt quá 1000 CCU, bạn sẽ cần chạy nhiều Server Node.js cùng lúc. Lúc này bạn phải cài đặt **Socket.io Redis Adapter** để các server có thể "nói chuyện" với nhau (Ví dụ: Player 1 ở Server A vẫn có thể bắn đạn trúng Player 2 ở Server B).
- **Matchmaking (Tìm trận):** Cần viết một Worker chạy ngầm để quét những người chơi đang tìm trận, so sánh điểm Rank (MMR) của họ và ghép cặp.

---

## 3. Frontend & UI (Nên nâng cấp)

> [!TIP]
> **Hiện tại:** Vanilla JS + HTML + CSS. Rất mượt cho Game Logic, nhưng khi bạn nhét thêm màn hình Đăng nhập, Cài đặt, Danh sách bạn bè, Lịch sử đấu, Bảng ngọc... file `script.js` sẽ dài hàng vạn dòng và không thể bảo trì nổi.

**Stack Đề Xuất:**
- **React.js** hoặc **Vue.js**: Dùng để quản lý các màn hình (Lobby, Leaderboard, History).
- **Phần Core Game (Vẽ đồ thị, Pháo hoa):** Vẫn giữ nguyên bằng Vanilla JS hoặc đưa vào một thẻ `<canvas>` để đảm bảo hiệu năng không bị ảnh hưởng bởi React Re-render.

---

## 4. Authentication & Authorization (Đăng nhập & Phân quyền)

Tự build hệ thống Auth (băm password, quên mật khẩu, verify email...) rất tốn thời gian.
**Stack Đề Xuất:**
- **Firebase Auth** hoặc **Supabase Auth**: Miễn phí cho quy mô nhỏ/vừa. Cho phép người dùng đăng nhập tức thì bằng **Google, Facebook, Github**. 
- Sau khi đăng nhập, Client sẽ nhận được một chuỗi `JWT Token`. Khi Client connect vào Socket.io, sẽ gửi kèm Token này. Server chỉ việc verify Token là biết chính xác ai đang chơi, chống được hack/fake tài khoản.

---

## 5. Hệ thống Rank (Xếp Hạng)

Đây là tính năng quan trọng nhất để giữ chân người chơi (Retention).

**Thuật toán đề xuất:**
- **Elo Rating System** (Giống cờ vua / LMHT): Thuật toán tính toán điểm cộng/trừ sau mỗi ván dựa trên chênh lệch trình độ. Nếu bạn thắng một người Rank cao, bạn cộng rất nhiều điểm. Nếu thua một người Rank thấp, bạn trừ rất nặng.
- **Quy trình luân chuyển:**
  1. Trận đấu kết thúc -> Server (chứ không phải Client) tổng hợp điểm số.
  2. Server tính toán sự thay đổi Elo.
  3. Server lưu Log vào `Sessions` MongoDB và Cập nhật điểm Elo vào `Players` MongoDB.
  4. Server broadcast kết quả cho cả 2 bên.

---

## Tóm tắt Lộ Trình (Roadmap) nếu bạn muốn Scale:

1. **Phase 1: Database Migration.** Chuyển toàn bộ logic đọc/ghi file hiện tại lên **MongoDB Cloud** (Dùng Atlas bản Free).
2. **Phase 2: Authentication.** Tích hợp Firebase Auth. Xoá tính năng tự do nhập tên, bắt buộc đăng nhập để lấy tên và Avatar.
3. **Phase 3: Redis & Matchmaking.** Thêm luồng "Tìm trận ngẫu nhiên" (Find Match) dựa trên Elo thay vì tự nhập Code phòng.
4. **Phase 4: Frontend Refactor.** Đưa UI hiện tại vào React để làm các tính năng phụ trợ (Lịch sử, Bảng xếp hạng) dễ dàng hơn.
