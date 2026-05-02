# Hướng dẫn Deploy Poly Apple lên Production

## 1. Trả lời câu hỏi: "Push lên Github rồi lấy URL có chơi được không?"

> [!WARNING]
> Câu trả lời là **KHÔNG**. 

Nếu bạn chỉ push code lên Github và dùng Github Pages (hoặc Vercel/Netlify tĩnh), người dùng **chỉ tải được giao diện** (HTML/CSS) nhưng **không thể tạo phòng hay chơi Multiplayer được**.

Lý do: Poly Apple có một "Bộ não" (Backend) nằm ở file `server.js` chạy bằng **Node.js** và dùng **Socket.io** để kết nối các người chơi với nhau. Github Pages không hỗ trợ chạy file `server.js` này.

Để mọi người chơi được, bạn cần một dịch vụ Hosting hỗ trợ **Node.js Runtime** để máy chủ luôn chạy 24/7 và xử lý kết nối Socket.

---

## 2. Lựa chọn dịch vụ Deploy (Khuyên dùng)

Hiện tại có 2 nền tảng xịn, dễ dùng, tự động hoàn toàn và **CÓ GÓI MIỄN PHÍ** cực kỳ phù hợp cho người chưa có kinh nghiệm setup:

1. **Render.com** (Dễ nhất, khuyên dùng)
2. **Railway.app**

Dưới đây là kế hoạch Step-by-Step deploy bằng **Render.com**.

---

## 3. Kế hoạch Deploy Step-by-Step (Dùng Render.com)

### Bước 1: Chuẩn bị Code trên Github
1. Đăng nhập Github và tạo một Repository mới (ví dụ: `poly-apple-game`).
2. Push toàn bộ source code hiện tại (đã được dọn dẹp các file rác) lên Repository này.
   - *Lưu ý:* Phải đảm bảo có file `package.json` và file `server.js`.
   - *Lưu ý 2:* Trong file `.gitignore` không được ignore thư mục `data/` nếu bạn muốn giữ data gốc, nhưng tốt nhất nên ignore để server tạo lại DB trống rỗng cho sạch sẽ.

### Bước 2: Tạo tài khoản Render
1. Truy cập [Render.com](https://render.com) và đăng nhập bằng tài khoản Github của bạn.
2. Tại màn hình Dashboard, bấm nút **"New"** -> Chọn **"Web Service"**.

### Bước 3: Kết nối với Github
1. Render sẽ yêu cầu quyền truy cập vào Github. Hãy cấp quyền cho nó đọc Repository `poly-apple-game` bạn vừa tạo.
2. Render sẽ tự động kéo (pull) source code từ Github về.

### Bước 4: Cấu hình Web Service
Render sẽ yêu cầu bạn điền một vài thông số (rất đơn giản):
- **Name:** `poly-apple` (hoặc tên bất kỳ).
- **Environment:** Chọn `Node`.
- **Build Command:** Nhập `npm install`. (Lệnh này bảo Render cài đặt các thư viện cần thiết như socket.io).
- **Start Command:** Nhập `npm start` (hoặc `node server.js`). Lệnh này sẽ khởi động bộ não của Game.
- **Instance Type:** Chọn gói **Free** (Miễn phí).

### Bước 5: Đợi Render "Nấu" (Build) & Khởi động
- Bấm **"Create Web Service"**.
- Màn hình sẽ hiện ra Terminal (giống y hệt cửa sổ dòng lệnh ở máy bạn). Render đang chạy `npm install` và sau đó chạy `node server.js`.
- Khi bạn thấy dòng chữ `Server running on port ...`, nghĩa là Game đã sống!

### Bước 6: Nhận URL và Chơi
- Ở góc trên cùng bên trái của Render, bạn sẽ thấy một đường link dạng `https://poly-apple.onrender.com`.
- **Đây chính là URL Production của bạn!**
- Bạn có thể gửi link này cho bạn bè, và ai cũng có thể vào tạo phòng, chơi với nhau bình thường.

---

## 4. Một số lưu ý cực kỳ quan trọng cho Production

> [!IMPORTANT]
> Vì hiện tại chúng ta đang dùng File JSON (`data/rooms.json`, `data/sessions.json`) để lưu database thay vì MongoDB xịn. Khi bạn dùng gói Free của Render, server có thể bị "ngủ đông" (sleep) nếu không có ai truy cập, và khi khởi động lại, **dữ liệu trên ổ đĩa có thể bị reset** (mất hết file json). 

- Với Poly Apple hiện tại (chỉ cần tạo phòng và chơi ngay), việc reset này **không ảnh hưởng gì tới trải nghiệm chơi**, vì phòng cũ vốn dĩ đã bị cronjob dọn dẹp.
- Tuy nhiên, nếu sau này bạn muốn làm tính năng **"Lịch sử thi đấu vĩnh viễn"** hoặc **"Bảng Xếp Hạng"**, bạn BẮT BUỘC phải chuyển qua dùng **MongoDB** theo lộ trình nâng cấp (Scaling Plan) trước đó.

Chúc bạn Deploy thành công! Nếu gặp vướng mắc ở bước nào cứ gọi tôi.
