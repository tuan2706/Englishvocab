# English Coach 🎯

Ứng dụng học tiếng Anh cá nhân — **"Biết từ chưa đủ, phải dùng được từ."**
Học từ → Flashcard → Nghe → Tự viết câu → **AI chấm & sửa lỗi** → Ôn tập.

Một file duy nhất `english-coach.html`. Không cần cài đặt, không tài khoản, không backend.
Dữ liệu lưu ngay trên máy bạn (localStorage).

---

## 1. Chạy thử ngay (không cần deploy)
Mở file `english-coach.html` bằng trình duyệt (Chrome/Safari/Edge) là chạy.

## 2. Đưa lên mạng bằng Netlify (khuyên dùng)
1. Vào https://app.netlify.com → kéo–thả file `english-coach.html` vào ô "Deploys".
   - **Mẹo:** đổi tên file thành `index.html` trước khi kéo, để mở link gọn hơn.
2. Netlify tạo link dạng `https://ten-ngau.netlify.app` → mở trên điện thoại.
3. Cập nhật sau này: vào tab **Deploys** của site đó, kéo file mới vào (đừng tạo site mới).

## 3. Cài như app trên điện thoại (PWA)
- **iPhone (Safari):** mở link → nút **Chia sẻ ⬆️** → **Thêm vào MH chính**.
- **Android (Chrome):** vào **Cài đặt** trong app → **Thêm vào màn hình chính**, hoặc menu Chrome.
- Mở từ icon ngoài màn hình → chạy toàn màn hình như app thật.

---

## 4. Bật AI chấm câu (miễn phí)
Mặc định app chấm câu bằng logic cơ bản. Bật Gemini để có nhận xét thông minh:

1. Lấy API key miễn phí tại **Google AI Studio**: https://aistudio.google.com/app/apikey
2. Trong app: **Tiến độ → Cài đặt → AI chấm câu (Gemini)** → dán key → **Lưu key**.
3. Xong. Mọi câu luyện sẽ được AI chấm 5 tiêu chí + sửa lỗi + gợi ý câu tự nhiên hơn.

> Key chỉ lưu trên máy bạn. Vì đây là app cá nhân nên để key ở trình duyệt là chấp nhận được.
> Gemini free tier có giới hạn số lượt/phút — nếu báo "hết lượt tạm thời", chờ vài phút hoặc app tự chấm bằng máy.

---

## 5. Sao lưu dữ liệu (quan trọng)
Dữ liệu ở localStorage → xóa trình duyệt / đổi máy là mất. Nên:
- **Tiến độ → Cài đặt → Xuất dữ liệu (backup)** → lưu file `english-coach-backup.json`.
- Khi cần khôi phục: **Nhập dữ liệu** → chọn file đó.

---

## 6. Tính năng đã có (MVP)
- **Home:** lời chào theo giờ, Continue learning, mục tiêu ngày, streak, quick actions.
- **Learn:** 6 chủ đề (Travel, Business, Environment, Technology, Work, Daily Life), tìm kiếm từ vựng.
- **Flashcard:** từ + IPA + phát âm (Web Speech API), lật xem nghĩa/ví dụ/collocations.
- **Practice:** viết câu dùng từ mục tiêu → chấm điểm (AI hoặc cục bộ) → sửa lỗi → làm lại.
- **Review:** ôn tập giãn cách (1/2/4/7/14 ngày), "Words to review" (từ hay sai).
- **Progress:** từ đã học, streak, điểm luyện, biểu đồ 7 ngày.
- **Settings:** mục tiêu ngày, sáng/tối, key AI, export/import, cài PWA.

## 7. Cấu trúc dữ liệu (dễ mở rộng)
Từ vựng nằm trong mảng `VOCAB` (trong `<script>`), mỗi từ đủ trường: `word, type, level,
meaningVi, definitionEn, pronunciation, exampleEn, exampleVi, collocations, targetExercise`.
Thêm từ mới = thêm một object vào mảng này. Chủ đề nằm trong mảng `TOPICS`.

---

*Ghi chú kỹ thuật:* Bản này là single-file (1 HTML) theo hướng tối giản, dễ sửa bằng Claude.
Nếu sau này muốn bản Vite/React/TypeScript + serverless để giấu API key (khi chia sẻ nhiều người),
có thể chuyển đổi — kiến trúc dữ liệu đã tách sẵn để việc đó dễ dàng.
