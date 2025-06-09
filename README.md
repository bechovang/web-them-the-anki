# web-them-the-anki

Tuyệt vời! Bạn hoàn toàn có thể xây dựng **chỉ frontend** (Frontend Only) để thực hiện việc:
- Nhập từ vựng.
- Gửi prompt đến Google Gemini API trực tiếp từ trình duyệt.
- Nhận kết quả và hiển thị dưới dạng text theo mẫu.

Việc không dùng backend sẽ giúp bạn **triển khai nhanh hơn**, không cần lo lắng về server, deploy backend, hoặc xử lý CORS phức tạp. Tuy nhiên, bạn cần lưu ý một số điểm quan trọng liên quan đến bảo mật và giới hạn API khi gọi từ phía client.

---

## ✅ Kế hoạch chi tiết cho Web Frontend Only

---

### 🎯 Mục tiêu
Xây dựng trang web chỉ dùng HTML/CSS/JavaScript (có thể dùng React nếu muốn), cho phép người dùng:

1. Nhập nhiều từ tiếng Anh (mỗi từ 1 dòng).
2. Gửi yêu cầu đến **Gemini API** trực tiếp từ frontend.
3. Hiển thị kết quả theo định dạng:  
   `Word;Pronunciation;Vietnamese;Synonyms;Antonyms;Example Sentence (EN + VI)`
4. Sao chép kết quả dễ dàng vào Anki hoặc file CSV.

---

## 🔐 CẢNH BÁO QUAN TRỌNG VỀ BẢO MẬT

> Vì bạn sẽ gọi Gemini API trực tiếp từ frontend, **API Key sẽ bị lộ** cho người dùng nếu không có biện pháp che giấu. Đây là **rủi ro bảo mật nghiêm trọng** vì ai cũng có thể lấy key của bạn để sử dụng, gây mất phí hoặc bị khóa tài khoản.

### Giải pháp thay thế an toàn hơn:
- Dùng **Google App Script** như proxy để gọi API → giữ kín key.
- Hoặc dùng **Cloudflare Worker / Vercel Edge Function / Firebase Function** để làm layer trung gian.

Nếu bạn vẫn muốn thử nghiệm với frontend-only trước, thì mình sẽ hướng dẫn cả cách **che giấu key tạm thời bằng cách đặt trong file `.env` và build static site**, nhưng lưu ý rằng nó **không phải là giải pháp bảo mật thật sự**.

---

## 🧱 CẤU TRÚC FRONTEND

```
anki-generator/
│
├── index.html          # Trang chính
├── style.css           # CSS tùy chọn
├── app.js              # Logic chính
├── .env                # Lưu trữ API_KEY (nếu build với Vite, ...) - KHÔNG hiệu quả tuyệt đối
└── ...
```

---

## 📦 Công nghệ đề xuất

| Thành phần | Công nghệ |
|----------|-----------|
| Ngôn ngữ | JavaScript (hoặc TypeScript) |
| Framework | Có thể dùng React hoặc chỉ thuần JS |
| Hosting | GitHub Pages, Netlify, Vercel,... |

---

## 🛠️ Chi tiết từng phần

---

### 1. Giao diện người dùng (index.html)

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Anki Word Generator</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <div class="container">
    <h1>Tạo dữ liệu học từ vựng Anki</h1>
    <textarea id="wordInput" placeholder="Nhập từ, mỗi từ trên 1 dòng..."></textarea>
    <button onclick="sendToGemini()">Gửi</button>

    <pre id="resultArea"></pre>
    <button onclick="copyResult()">Sao chép</button>
  </div>
  <script src="app.js"></script>
</body>
</html>
```

---

### 2. Logic xử lý (app.js)

```javascript
const API_KEY = "YOUR_GEMINI_API_KEY"; // RỦI RO: Key bị lộ
const API_URL = "https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent";

const PROMPT_TEMPLATE = `
Xin chào, hãy tạo bảng thông tin từ vựng tiếng Anh với các cột sau:
- Word: từ tiếng Anh
- Pronunciation: phiên âm chuẩn quốc tế
- Vietnamese: nghĩa tiếng Việt ngắn gọn
- Synonyms: các từ đồng nghĩa chính
- Antonyms: các từ trái nghĩa (nếu có)
- Example Sentence (EN + VI): câu ví dụ bằng tiếng Anh và bản dịch tiếng Việt. Trong câu ví dụ, hãy bôi đậm từ chính tiếng Anh và từ tiếng Việt tương ứng bằng "...".

Dữ liệu trả về dưới dạng text, các cột cách nhau bởi dấu ";", mỗi hàng là 1 từ. Nếu từ có nhiều nghĩa thì ghi đủ các nghĩa đó, mỗi nghĩa là 1 hàng riêng biệt.

Ví dụ:
run;/rʌn/;chạy;noun: chạy, động từ: di chuyển nhanh;stay still;"She likes to run in the morning." ("Cô ấy thích đi bộ buổi sáng.")

Hãy xử lý các từ sau:
`;

async function sendToGemini() {
  const input = document.getElementById("wordInput").value.trim();
  if (!input) return alert("Vui lòng nhập từ!");

  const fullPrompt = PROMPT_TEMPLATE + "\n\n" + input;

  const response = await fetch(`${API_URL}?key=${API_KEY}`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      contents: [{ parts: [{ text: fullPrompt }], role: "user" }]
    })
  });

  const data = await response.json();
  const resultText = data.candidates?.[0]?.content?.parts?.[0]?.text || "Lỗi!";
  document.getElementById("resultArea").textContent = resultText;
}

function copyResult() {
  const text = document.getElementById("resultArea").textContent;
  navigator.clipboard.writeText(text).then(() => alert("Đã sao chép!"));
}
```

---

## 🧪 Test thử

Bạn có thể test với các từ sau:
```
light
book
run
```

---

## 🚀 Triển khai

### Cách triển khai đơn giản:
- Dùng [GitHub Pages](https://pages.github.com/)
- Hoặc [Netlify Drop](https://www.netlify.com/drop/)
- Hoặc [Vercel Drag & Drop](https://vercel.com/)

### Cách triển khai tốt hơn (che giấu key):
- Dùng **Google Apps Script** làm proxy API
- Hoặc **Cloudflare Workers**
- Hoặc **Firebase Functions**

=> Mình có thể giúp bạn viết code cho từng phương án này nếu bạn muốn nâng cấp bảo mật.

---

## ✅ Kết luận

Bạn đã có kế hoạch chi tiết và đầy đủ code mẫu để bắt đầu xây dựng trang web frontend-only để gửi từ vựng tới Gemini API và nhận kết quả theo định dạng Anki. Với vài file đơn giản, bạn có thể chạy ngay trên máy tính cá nhân hoặc host miễn phí.

---

Nếu bạn muốn:
- Code mẫu đầy đủ zip để tải về.
- Hướng dẫn triển khai trên GitHub Pages.
- Cách dùng Google Apps Script làm proxy để bảo vệ API KEY.
- Thêm chức năng upload file hoặc export CSV.

👉 Hãy nói rõ nhu cầu nhé! Mình sẽ hỗ trợ cụ thể.
