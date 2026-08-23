# 📘 English Coach — Tài liệu tổng hợp dự án

> **Cách dùng file này:** Đính kèm file này (`PROJECT_ENGLISH_COACH.md`) **cùng với** file app hiện tại (`english-coach.html`) vào cuộc trò chuyện mới. Claude sẽ đọc file này để hiểu toàn bộ bối cảnh, quy ước, và lịch sử phát triển — sau đó đọc `english-coach.html` để lấy code mới nhất — rồi có thể tiếp tục chỉnh sửa ngay mà không cần bạn giải thích lại từ đầu.
>
> Nếu chỉ đính kèm mỗi file này (không có `.html`), Claude vẫn hiểu được toàn bộ bối cảnh và quyết định kỹ thuật, nhưng cần bạn gửi kèm file `.html` hiện tại để có code thật.

---

## 1. Tổng quan dự án

**English Coach** là một app học tiếng Anh dạng **PWA single-file HTML** (1 file `.html` duy nhất chứa toàn bộ HTML/CSS/JS, không build step, không backend), dành riêng cho một người dùng (không phải sản phẩm SaaS nhiều người dùng).

- **Đối tượng dùng:** kỹ sư kinh doanh (technical sales) ngành thiết bị quan trắc môi trường, học tiếng Anh công việc thực tế cho ngành của mình.
- **Người phát triển:** không rành kỹ thuật ("vibe coding") — mô tả yêu cầu bằng tiếng Việt (đôi khi qua file Word đính kèm), Claude thực hiện toàn bộ code.
- **Ngôn ngữ giao diện:** 100% tiếng Việt.
- **Triển khai:** kéo-thả file `.html` lên **Netlify** (tab Deploys của site đã có sẵn). Không có quy trình build/deploy phức tạp.
- **Quy mô hiện tại:** ~816 từ vựng, file `.html` ~860KB.

### Nguyên tắc làm việc bất di bất dịch
Mỗi khi có yêu cầu chỉnh sửa mới:
1. **KHÔNG viết lại app từ đầu.** Đây luôn là yêu cầu UPDATE, không phải rewrite.
2. **KHÔNG xóa tính năng hiện có, KHÔNG làm mất dữ liệu vocabulary, KHÔNG đổi cấu trúc DATA nếu không cần thiết.**
3. Nếu đổi schema dữ liệu → **luôn phải backward-compatible** (merge dữ liệu cũ, không reset).
4. Đọc kỹ code hiện tại trước khi sửa (dùng `grep`/`view`), sửa tối thiểu — không đổi tên hàm/biến nếu không cần.
5. Sau khi sửa: **luôn kiểm tra cú pháp** (đếm ngoặc `{}`, chạy `new Function()` trên từng khối `<script>` bằng regex non-greedy — file có ≥2 thẻ `<script>`), **luôn test bằng Playwright** trước khi giao file (xem mục 8).
6. Luôn xuất file cuối cùng ra `/mnt/user-data/outputs/english-coach.html` và gọi `present_files`.

---

## 2. Kiến trúc kỹ thuật

**Stack:** HTML thuần + React qua CDN (không JSX build, dùng vanilla JS render-string pattern — xem mục 4) + CSS thuần (custom properties làm design token) + Gemini API (gọi trực tiếp từ client bằng `fetch`, không proxy).

**Kiến trúc app shell (mobile-first, đã redesign 2 lần):**
```
<body>
  <script> ... phát hiện bàn phím (visualViewport) ... </script>
  <div class="app">              ← full-screen trên mobile, khung ~430px giữa màn trên desktop (≥470px)
    <main class="main" id="main">  ← VÙNG CUỘN DUY NHẤT của toàn app
    <div class="fab" id="fab">     ← nút + nổi, position:absolute trong .app
    <nav class="nav">              ← bottom tab bar, position:absolute trong .app, translucent blur
    <div class="overlay">/<div class="sheet">  ← bottom sheet dùng chung (Cài đặt, Ghi chú, Tra từ...)
  </div>
</body>
```

**⚠️ Bài học đắt giá về mobile viewport/keyboard (đã qua nhiều vòng sửa lỗi):**
- **KHÔNG dùng JS đo `visualViewport`/`innerHeight` để liên tục resize `.app`/`body`/`.main` height.** Đây là nguồn gốc của rất nhiều bug "khoảng đen", "giật", "nhảy layout" đã gặp phải. Giải pháp cuối cùng (đang dùng): **CSS thuần `height:100vh;height:100svh;height:100dvh;overflow:hidden`** cho `body` và `.app` — để trình duyệt tự lo, JS không can thiệp chiều cao.
- **JS chỉ được dùng để PHÁT HIỆN bàn phím mở/đóng** (so `visualViewport.height` với `window.innerHeight`, chênh >120px = bàn phím mở) → bật/tắt class `body.kb-open` → CSS lo phần còn lại (`body.kb-open .nav{display:none}`, `body.kb-open .fab{display:none!important}` — **chú ý `!important`** vì JS khác trong app có set `fab.style.display` inline, cần `!important` để thắng).
- **KHÔNG dùng `scrollIntoView()` tự động toàn cục khi focus input.** Đã từng thêm global `focusin` handler tự cuộn — gây đúng hiệu ứng "giật" mà người dùng phàn nàn. Đã bỏ hoàn toàn.
- **`.main` luôn là vùng cuộn DUY NHẤT** (`overflow-y:auto`), `body`/`html`/`.app` đều `overflow:hidden`. Không để nhiều tầng cùng scroll.
- **`env(safe-area-inset-top)` và `env(safe-area-inset-bottom)`** bắt buộc phải có trong padding của `.main` — thiếu cái này là nguyên nhân chữ bị đè lên status bar/tai thỏ.
- **`theme-color` (meta tag) và màu nền trong `manifest.json` (base64-encode trong `<link rel="manifest">`) phải khớp CHÍNH XÁC** với `--bg` CSS variable thực tế — lệch màu dù rất nhỏ sẽ lộ ra thành "khoảng đen" khi có safe-area hoặc splash screen PWA.
- **"Tra từ" (dictionary lookup) dùng bottom sheet** (`showSheet()`/`closeSheet()`), KHÔNG phải panel inline trong luồng cuộn chính — vì panel inline gây nhảy scroll khi mở/đóng. Sheet mở/đóng không đụng tới scroll position của `.main` phía sau.

**Desktop (`@media(min-width:470px)`):** `.app` có `max-width:430px`, `border-radius:28px`, `box-shadow`, margin để tạo cảm giác "app preview" giữa màn hình. Trên mobile (mặc định, <470px) — full-screen tuyệt đối, không radius/shadow/margin.

---

## 3. Cấu trúc dữ liệu (localStorage)

**Key:** `english-coach-v1`. Cơ chế load an toàn — merge dữ liệu cũ vào default mới, KHÔNG BAO GIỜ reset:
```js
function load(){
  try{ const raw=localStorage.getItem(KEY); if(!raw) return defaultData();
    return { ...defaultData(), ...JSON.parse(raw) }; }
  catch(e){ return defaultData(); }
}
```
→ Khi thêm field DATA mới, chỉ cần thêm vào `defaultData()`, dữ liệu cũ của user tự động được giữ nguyên và field mới tự có giá trị mặc định. **Đây là pattern bắt buộc phải giữ khi thêm field mới.**

### Schema `DATA` hiện tại (`defaultData()`):
```js
{
  learned: {},              // { vocabId: {ts, box, firstDay} } — firstDay = ngày học ĐẦU TIÊN (không đổi khi học lại,
                             //   dùng để lịch "từ đã học theo ngày" nhóm đúng theo ngày gốc)
  reviewQueue: {},          // { vocabId: nextReviewDate } — spaced repetition
  dailyGoal: 5,
  progressToday: { date, count },
  streak: { count, lastDay },
  sentencesPracticed: 0,
  practiceScores: [],       // [{ts, score}]
  mistakes: {},             // { vocabId: count }
  myWords: [],              // [{id, word, meaningVi, ts}] — từ tự lưu (từ Gợi ý từ / Tra từ)
  notes: [],                // [{id, ts, kind:'fix'|'manual', bad, good, tag/category, reason/memoryTip,
                             //   example, sentence, word, myNote, text}] — sổ tay lỗi + ghi chú cá nhân
  generatedExercises: {},   // { vocabId: [{promptVi,targetPhrase,difficulty,context,grammarFocus,ts}] }
                             //   — cache đề "Khó" do AI sinh, tránh gọi AI lặp lại + chống trùng câu
  theme: 'dark',
  geminiKey: '',
  geminiModel: '',
}
```

### Schema `VOCAB` (từng từ, trong mảng `const VOCAB = [...]`, ~816 phần tử):
```js
{
  id, topic, word, type, level,      // level = CEFR: A1/A2/B1/B2
  meaningVi, definitionEn, pronunciation,
  exampleEn, exampleVi, collocations: [],
  targetExercises: [                 // 3 phần tử cố định: medium/good/hard (Câu 1/2/3, 🟢🟡🔴)
    { promptVi, targetPhrase, level }
  ]
  // Từ RẤT CŨ có thể chỉ có `targetExercise` (số ít, 1 object) thay vì `targetExercises` (mảng) —
  // hàm getExercises(v) tự fallback: `Array.isArray(v.targetExercises)? v.targetExercises : (v.targetExercise?[v.targetExercise]:[])`
}
```
- **Topics (6):** travel(140), business(136), environment(133), technology(143), work(128), daily(136). ID prefix: `trv/biz/env/tec/wrk/day`.
- Toàn bộ 816 từ **tự biên soạn** (không copy từ nguồn có bản quyền).

---

## 4. Kiến trúc render (không dùng React JSX build)

App dùng pattern "render-string": mỗi màn hình là 1 hàm JS trả về chuỗi HTML, gán vào `#main.innerHTML`. State toàn cục là object `state = {...}`. Mọi tương tác gọi hàm JS qua `onclick="..."` inline, hàm đó sửa `state`/`DATA` rồi gọi `render()`.

**Các hàm/khối chính cần biết khi sửa code:**
| Vùng | Vai trò |
|---|---|
| `VIEWS = {home, learn, topic, flash, review, progress, practice, result, notes}` | Map tab → hàm render màn hình |
| `render()` | Re-render `#main`, cập nhật `#nav`, ẩn/hiện FAB theo `HIDE_FAB` |
| `state.fx[vocabId][exIdx]` | State luyện câu inline trong flashcard: `{input, result, loading, error, curEx, exLoading, hints, ...}` |
| `getExercises(v)` | Trả về mảng 3 đề cố định (medium/good/hard) của 1 từ, backward-compat |
| `fxCurrentEx(v, idx)` | Đề ĐANG hiển thị thực sự = `curEx` (nếu đã đổi/AI-sinh) hoặc đề cố định — **luôn dùng hàm này thay vì lấy trực tiếp từ `getExercises()[idx]`** ở mọi nơi cần biết đề hiện tại |
| `buildPrompt(sentence, vocab, ex)` | Prompt chấm điểm gửi Gemini (xem mục 5) |
| `evaluateWithGemini()` / `callGemini()` | Luồng chấm điểm chính — **KHÔNG được đổi trừ khi thật sự cần** |
| `generateExerciseWithGemini()` / `callGeminiRaw()` / `generateHardExercise()` | Luồng sinh đề "Khó" mới — **tách riêng hoàn toàn** khỏi luồng chấm điểm để không rủi ro logic cũ |
| `showSheet(html)` / `closeSheet()` | Bottom sheet dùng chung (Cài đặt, Ghi chú, Tra từ, sửa ghi chú...) |
| `DATA`, `save()`, `load()`, `defaultData()` | Quản lý localStorage |

---

## 5. Tích hợp AI (Gemini)

- **Model fallback list:** `GEMINI_MODELS = ['gemini-3.5-flash-lite','gemini-2.5-flash-lite','gemini-3.5-flash','gemini-2.5-flash']` — ưu tiên Flash-Lite (rẻ/nhanh), model đã chạy được lần trước lưu vào `DATA.geminiModel` và thử trước.
- **`evaluateWithGemini(sentence, vocab, ex)`**: thử lần lượt các model, backoff `[0,2000,5000]ms` khi gặp 429 theo phút, phân biệt rate-limit theo PHÚT (tự retry được) và theo NGÀY (`RATE_LIMIT_DAY`, báo user chờ mai).
- **Schema JSON trả về (đã chuẩn hoá key-word style, dễ hiểu cho người mới):**
  ```json
  {
    "score": 90, "meaningScore":95, "grammarScore":88, "vocabularyScore":92, "naturalnessScore":90,
    "targetWordUsed": true, "isAcceptable": true,
    "originalSentence": "...", "correctedSentence": "...",
    "fixes": [
      { "category": "PREPOSITION", "bad": "interested on", "good": "interested in",
        "memoryTip": "interested IN", "example": "I'm interested in AI." }
    ],
    "feedback": ["..."], "explanationVi": "...", "moreNaturalVersion": "..."
  }
  ```
  - `category`: từ khóa IN HOA tiếng Anh (GRAMMAR/PREPOSITION/ARTICLE/TENSE/PLURAL/WORD ORDER/COLLOCATION/VOCABULARY/NATURALNESS/TARGET WORD/SPELLING).
  - `memoryTip`: ≤8 từ, đời thường, **cấm thuật ngữ ngữ pháp học thuật** ("phân từ", "mệnh đề quan hệ"...).
  - **Tối đa 3 lỗi/lần chấm** (ưu tiên: sai nghĩa → ngữ pháp cơ bản → từ mục tiêu → collocation → tự nhiên).
  - Parse có **backward-compat**: chấp nhận cả field cũ `tag`/`reason`/`bestVersion` lẫn field mới `category`/`memoryTip`/`moreNaturalVersion`.
- **Sinh đề "Khó" đa dạng** (tách biệt hoàn toàn khỏi luồng chấm điểm, xem mục 6.9): `callGeminiRaw(model, promptText)` là hàm generic gọi Gemini với prompt tuỳ ý — **KHÔNG dùng chung với `callGemini`** để tránh rủi ro ảnh hưởng logic chấm điểm.
- **Google Translate free** (`gtranslate()`, dùng cho tính năng Tra từ) — endpoint công khai `translate.googleapis.com`, dự phòng MyMemory API, không cần key riêng.

---

## 6. Lịch sử tính năng đầy đủ (theo thứ tự phát triển)

1. **Vocab 316→816 từ**: sinh bằng Python script theo topic, dedup, rebalance CEFR (A1:100/A2:150/B1:150/B2:100).
2. **Gemini rate-limit mitigation**: model fallback + auto-retry backoff + phân biệt rate-limit phút/ngày.
3. **Ghi chú lỗi (Notes)**: nút "📌 Lưu" trên mỗi lỗi AI → mở sheet cho **tự viết ghi chú cá nhân** (`myNote`, không bắt buộc, khác với gợi ý AI) → hiển thị nổi bật trong màn Ghi chú + Lịch, có nút sửa lại. Ghi chú tay tự do cũng có, nhóm theo ngày (Hôm nay/Hôm qua/ngày cụ thể).
4. **Flashcard random, không lặp từ đã học**: "Học từ mới" → lọc bỏ từ đã học (`DATA.learned`) + `shuffle()`. Bấm từ cụ thể (từ search/list) vẫn mở đúng từ đó, không bị random.
5. **Câu 3 (Khó) nâng cấp lên mức IELTS**: 816 prompt Câu 3 viết lại thành câu phức dài (~26 khung câu học thuật/đời sống xoay vòng), Câu 1/2 giữ nguyên đơn giản.
6. **Google Translate lookup + lưu "Từ của tôi"**: tra nhanh Việt↔Anh ngay trong lúc luyện câu, tự nhận diện chiều dịch theo dấu tiếng Việt, lưu từ vào thư viện cá nhân.
7. **Lịch "Từ đã học theo ngày"** (thay thế hẳn màn Ôn tập cũ): lưới T2-CN kiểu iOS Calendar, badge số từ/ngày, bấm ngày → danh sách từ (nghĩa + loại từ + ghi chú nếu có) → bấm từ → mở rộng xem ghi chú + nút Luyện lại. Nút "🔁 Ôn hôm nay" (spaced repetition cũ) giữ lại gọn ở đầu trang.
8. **Mobile full-screen + xử lý bàn phím** (nhiều vòng lặp sửa lỗi, xem mục 2 để biết giải pháp cuối): từ dùng `position:fixed` → `100vh` JS-driven → cuối cùng chốt ở **CSS thuần `dvh/svh` + JS chỉ detect bàn phím**.
9. **Redesign giao diện phong cách Apple/iOS** (visual-only, không đổi chức năng): system font (`-apple-system,...`), radius mềm hơn (card 16-20px, button 12-14px thay vì bo tròn quá mức), shadow rất nhẹ, bottom tab bar translucent blur, list row kiểu Apple Settings (icon-title-subtitle-chevron) thay vì card lớn cho danh sách topic.
10. **AI feedback redesign "key-word style"**: từ đoạn văn dài → thẻ ngắn (category chip → sai/đúng → 💡 mẹo nhớ cực ngắn → ví dụ mini), tối đa 3 lỗi ưu tiên.
11. **Hệ thống đề luyện tập đa dạng cho mức Khó** (mới nhất): nút "🔄 Câu khác" chỉ hiện ở Câu 3 (Khó) → ưu tiên random trong pool có sẵn (đề cố định + cache), gọi AI sinh thêm câu mới khi pool ít hoặc ngẫu nhiên tăng đa dạng, chống lặp bằng so khớp độ tương đồng từ vựng + retry tối đa 2 lần, cache tối đa 20 câu/từ, level-aware theo CEFR (Hard của A2 khác Hard của B2).

---

## 7. Design system (CSS tokens hiện tại)

```css
/* Dark theme (mặc định) */
--bg:#0b1020; --surface:#161d2e; --surface-2:#1f2739;
--border:rgba(255,255,255,.08); --border-strong:rgba(255,255,255,.15);
--ink:#f5f7fa; --muted:#8b93a7;
--brand:#6366f1; --brand-2:#7c3aed;   /* Indigo/Violet — brand color GIỮ NGUYÊN xuyên suốt dự án */
--success:#32d583; --danger:#f97066; --warn:#fdb022;
--r-card:20px; --r-btn:14px;          /* Radius kiểu iOS — không bo tròn quá mức */
--sh-card: 0 1px 2px rgba(0,0,0,.18), 0 6px 20px -12px rgba(0,0,0,.45);  /* Shadow rất nhẹ */
```
- **Light theme:** `html.light{...}` — nền `#f2f4f9`, surface trắng, shadow/border tương ứng nhẹ.
- **Font:** `-apple-system, BlinkMacSystemFont, "SF Pro Text/Display", "Helvetica Neue", "Open Sans"/"Poppins" (fallback), sans-serif` — ưu tiên system font iOS, không tải font nặng ngoài Google Fonts fallback đã có sẵn.
- **Spacing scale:** 4/8/12/16/20/24/32px. Padding nội dung mobile ~16-20px. Touch target ≥44px.
- **Bottom nav:** `backdrop-filter:blur(24px) saturate(180%)`, translucent, active state nhẹ (không quá đậm).

---

## 8. Quy ước testing (Playwright)

- **Chromium sẵn có:** `/opt/pw-browsers/chromium-1194/chrome-linux/chrome`, luôn `args:['--no-sandbox']`.
- **Cài mỗi phiên:** `npm install playwright --no-save` (không lưu vào package.json vì không có).
- **Kích thước bắt buộc test mobile:** 375, 390, 393, 402, 412, 430px (height 844, `isMobile:true, hasTouch:true, deviceScaleFactor:2-3`).
- **Desktop:** 1280px, 1440px.
- **Mock Gemini:** `page.route('**/generativelanguage.googleapis.com/**', ...)` trả JSON đúng schema mục 5.
- **Mock Google Translate:** `page.route('**/translate.googleapis.com/**', ...)` trả `[[[translated,orig,null,null]]]`.
- **Serve:** `python3 -m http.server PORT & sleep 1; node test.mjs; kill $SRV`.
- **Kiểm tra cú pháp bắt buộc trước khi giao file** (file có nhiều thẻ `<script>`, phải dùng regex **non-greedy**):
  ```js
  const re=/<script>([\s\S]*?)<\/script>/g; let m;
  while((m=re.exec(html))){ try{ new Function(m[1]); }catch(e){ /* báo lỗi */ } }
  ```
- **Luôn xác nhận sau mỗi lần sửa:** VOCAB vẫn = 816, không có lỗi console (`page.on('pageerror', ...)`), không scroll ngang (`scrollWidth > clientWidth`), dữ liệu cũ (`streak/geminiKey/learned/notes`) còn nguyên sau `reload()`.
- **Dọn dẹp trước khi xuất:** `rm -f *.mjs *.png; rm -rf node_modules`, copy vào `/mnt/user-data/outputs/english-coach.html`, gọi `present_files`.

---

## 9. Sở thích & quy trình làm việc của người dùng

- **Giao tiếp:** kỹ thuật nhưng dễ hiểu, gọn gàng; hướng dẫn "vibe coding" chi tiết vì người dùng non-tech.
- **"tiếp nhé"/"Continue"** = làm tiếp không cần hỏi lại, tự quyết định phương án tốt nhất.
- **Thường gửi yêu cầu qua file Word** (`.docx`) — đọc bằng `pandoc -t markdown file.docx`, trích ảnh minh hoạ bằng `unzip` vào `word/media/` nếu có.
- **Ưu tiên sửa an toàn, tối thiểu** hơn là giải pháp "đẹp" nhưng rủi ro — đặc biệt với vocabulary và AI evaluation logic.
- **Deploy:** luôn nhắc kéo file `.html` mới lên Netlify (tab Deploys), và nhắc xóa cache trình duyệt/thêm lại icon màn hình chính khi test trên điện thoại (đã từng bị nhầm là "chưa sửa" do cache cũ).
- **Báo cáo sau khi hoàn thành:** liệt kê rõ đã sửa gì, ở đâu, xác nhận build/test status, nêu vấn đề còn lại nếu có.

---

## 10. Việc cần làm khi mở lại project (checklist nhanh)

1. `cp` file `.html` người dùng gửi (hoặc bản mới nhất) vào workspace, đọc qua cấu trúc bằng `grep`/`view` — **đừng giả định, luôn đọc code thật trước khi sửa**.
2. Nếu có file Word yêu cầu mới → đọc bằng `pandoc`, trích ảnh nếu có.
3. Xác định thay đổi tối thiểu cần thiết, tránh động vào: `VOCAB`, `evaluateWithGemini`/`callGemini`, cấu trúc `DATA` (trừ khi thêm field mới theo pattern backward-compat).
4. Sửa code, kiểm tra cú pháp, test bằng Playwright theo mục 8.
5. Xuất file, `present_files`, báo cáo ngắn gọn bằng tiếng Việt.
6. Nếu có thay đổi lớn về tính năng/kiến trúc → **cập nhật lại file tài liệu này** và gửi kèm bản mới.

---

*Tài liệu này được tạo để Claude có thể tiếp tục phát triển English Coach ở bất kỳ cuộc trò chuyện mới nào mà không mất bối cảnh. Cập nhật lần cuối: sau khi hoàn thành hệ thống đề luyện tập đa dạng mức Khó + redesign mobile shell theo yêu cầu "full-screen app thực sự".*
