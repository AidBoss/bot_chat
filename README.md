# Telegram AI bot

Bot Telegram chat bằng Gemini (free), chạy serverless trên Vercel, nhớ lịch sử chat bằng Upstash Redis, và tự gửi tin tan ca vào nhóm theo lịch QStash.

Mọi thứ nằm trong một file [api/bot.py](api/bot.py) (endpoint `/api/bot`), chỉ dùng thư viện chuẩn Python.

## Biến môi trường (đặt trên Vercel)

| Biến | Bắt buộc | Ý nghĩa |
|---|---|---|
| `TELEGRAM_TOKEN` | ✔ | Token bot lấy từ @BotFather |
| `BOT_USERNAME` | ✔ | Username của bot, ví dụ `may_ai_bot` |
| `WEBHOOK_SECRET` | ✔ | Chuỗi bí mật tự đặt (chỉ gồm `A-Z a-z 0-9 _ -`), tạo bằng `openssl rand -hex 32` |
| `GEMINI_API_KEY` | ✔ | Key lấy ở https://aistudio.google.com/apikey |
| `GEMINI_MODEL` | | Mặc định `gemini-flash-latest` |
| `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN` | ✔ | Lấy ở Upstash Console → Redis → tab REST API |
| `QSTASH_CURRENT_SIGNING_KEY`, `QSTASH_NEXT_SIGNING_KEY` | ✔ | Lấy ở Upstash Console → QStash → Signing Keys |
| `ALLOWED_IDS` | | Thêm user id / chat id được dùng bot, cách nhau bằng dấu phẩy. Người tạo và nhóm mặc định `-1003719756588` luôn có quyền, không cần điền |
| `CREATOR_NAME`, `CREATOR_ALIASES` | | Tên người tạo và các biến thể cần bảo vệ (mặc định: Đức Anh, Đinh Đức Anh, Đinh Anh, anhdd, Anh Đức, người tạo, chủ bot) |
| `GIPHY_API_KEY` | | Key free lấy ở https://developers.giphy.com. Không đặt thì bot không gửi GIF |
| `MAX_HISTORY` | | Số lượt chat bot nhớ cho mỗi chat (mặc định `100`; mỗi lượt là 1 tin gọi bot hoặc 1 câu bot trả lời) |
| `SYSTEM_PROMPT` | | Ghi đè tính cách mặc định (bot "Tèo": thật thà, ngơ ngơ, cà khịa có giới hạn) |
| `OFFWORK_PROMPT` | | Bot nên viết gì vào tin tan ca |

## Triển khai

1. **Tạo bot:** trên Telegram, nhắn @BotFather, gõ `/newbot`. Sau đó gõ `/setprivacy` → chọn bot → `Disable` để bot thấy được các tin @mention trong nhóm. Nếu bot đã ở trong nhóm từ trước thì xoá ra và thêm lại.
2. **Deploy lên Vercel:** chạy `npx vercel --prod` trong thư mục này (hoặc import repo trên vercel.com). Khai báo đủ biến môi trường ở trên. Nhóm mặc định và người tạo đã ghi sẵn trong code, nên có thể bỏ trống `ALLOWED_IDS`.
3. **Nối Telegram với Vercel:**
   ```bash
   curl "https://api.telegram.org/bot$TELEGRAM_TOKEN/setWebhook" -d "url=https://<app>.vercel.app/api/bot" -d "secret_token=$WEBHOOK_SECRET"
   ```
4. **Lấy id:** nhắn riêng cho bot, rồi @mention bot trong nhóm. Bot sẽ trả lời kèm `user id` và `chat id`. Chỉ cần làm bước này khi thêm nhóm khác ngoài nhóm mặc định: điền chat id của nhóm đó vào `ALLOWED_IDS`, rồi **redeploy** (Vercel chỉ nhận biến mới sau khi deploy lại).
5. **Hẹn giờ:** vào Upstash Console → QStash → Schedules, tạo 2 lịch, cùng Destination `https://<app>.vercel.app/api/bot`. Cron tính theo giờ UTC (giờ Việt Nam trừ 7 tiếng).

   | Lịch | Cron (UTC) | Body |
   |---|---|---|
   | Tan ca 17h34, Thứ 2 → Thứ 6 | `34 10 * * 1-5` | `{"job":"offwork"}` |
   | Tan ca 17h04, Thứ 7 | `4 10 * * 6` | `{"job":"offwork"}` |

   Tin tan ca luôn gửi vào nhóm mặc định `-1003719756588`. Muốn thêm nhóm khác thì gõ `/tanca on` trong nhóm đó, và nhóm đó phải nằm trong `ALLOWED_IDS`.

   Lịch thứ 7 được gọi mọi thứ 7, nhưng code chỉ gửi tin vào **thứ 7 đi làm**: thứ 7 đầu tháng (nếu rơi vào mùng 1, 2, 3 thì dời sang thứ 7 kế tiếp) và thứ 7 cuối tháng.

## Dùng

- Chat riêng với bot: bot trả lời mọi tin.
- Trong nhóm: @mention bot hoặc reply tin của bot. Muốn bot đọc ảnh/file thì gửi kèm caption có @mention.
- **Lệnh của bot chỉ người tạo dùng được.** Người khác gõ lệnh của bot thì bot chỉ đáp "Biết cái gì mà nghịch, đi ra ngoài nắng mà chơi cho mát 😎", không chạy lệnh và không gọi Gemini.
- Lệnh không có trong bot (gõ bậy, lệnh của bot khác…) thì bot im lặng, ai gõ cũng vậy.
- `/nick <biệt danh>`: **reply vào tin của một người** rồi gõ lệnh để đặt biệt danh (gõ `/nick` không kèm chữ để xoá). Chỉ người tạo (`CREATOR_ID`, ghi cứng trong code) được đặt. Bot sẽ gọi người đó bằng biệt danh, hiểu khi ai đó nhắc tới biệt danh, và tag được đúng người.
- `/nicks`: xem danh sách biệt danh.
- `/xoagif` hoặc `/xoasticker`: **reply vào một GIF hoặc sticker** rồi gõ lệnh để cấm nó. Bot xoá tin đó ngay, và xoá tiếp mỗi khi có người gửi lại đúng GIF/sticker này. `/boxoagif` để bỏ cấm tất cả. Chỉ người tạo dùng được. Bot cần là **admin có quyền Xoá tin nhắn** trong nhóm.
- `/tanca on` / `/tanca off`: bật/tắt tin tan ca cho nhóm hiện tại. Chỉ người tạo được bật/tắt. Gõ `/tanca` để xem nhóm đang bật hay tắt. Nhóm mặc định luôn bật, không tắt được.
- `/reset`: xoá lịch sử chat.
- Chống spam: người nào gọi bot quá 10 tin trong 30 giây thì bot báo một lần rồi im lặng với người đó 2 phút. Muốn đổi mức này thì sửa `RATE_LIMIT`, `RATE_WINDOW`, `COOLDOWN` trong code. Lịch sử cũng tự xoá sau 7 ngày không chat.

## Kiểm tra

```bash
python3 test_bot.py
```
