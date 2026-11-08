# AutoBot - MTProto 4 tầng (Push + Raw + Active Difference)

**Luồng nhận mới:** socket TCP MTProto thường trực do Telethon quản lý, kết hợp đối chiếu `GetChannelDifference` cho đủ 22 kênh đang bật. Đây **không phải** WebSocket hay HTTP polling; các RPC Difference chạy tách khỏi handler.

```text
Telegram DC
    │  kết nối TCP MTProto (Telethon / ConnectionTcpAbridged)
    ▼
NewMessage / MessageEdited / Raw [đăng ký trước khi client.start()]
    │ kiểm tra channel ID, tuổi tin, chống lặp (RAM) - KHÔNG await
    ▼
Bounded Priority RAM Queue (INGEST_QUEUE_MAXSIZE, mặc định 128)
    │ ưu tiên: spoiler → text → image → video; sau đó priority của kênh
    │ đầy: thay tin cũ cùng/thấp ưu tiên, hoặc từ chối tin mới thấp ưu tiên
    ▼
Message Workers (MESSAGE_WORKERS)
    ├── text/spoiler -> trích xuất mã
    └── media -> download timeout -> OCR ảnh/video có giới hạn worker/queue
                                  │
                                  ▼
                      RAM queue theo website
                                  │
                                  ▼
                         Site workers + khóa tài khoản/tab
                                  │
                                  ▼
                         Playwright/HTTP -> submit -> kết quả
```

## Tích hợp

- `mtproto_push.py`: bộ nối MTProto Push/Raw sang RAM. Callback không tải media, OCR hay gửi mã.
- `mtproto_channel_diff.py`: prewarm peer/PTS và vòng RPC Difference giới hạn cho các kênh đang bật.
- `tg_ingest.py`: tiếp nhận đồng bộ với `put_nowait`, lọc kênh/tuổi, chống trùng tin/tin sửa trong RAM.
- `ram_backpressure.py`: queue có giới hạn, ưu tiên và thống kê bỏ tin khi quá tải.
- `main_script.py`: đăng ký handler **trước** `client.start()`; log gộp thay vì log INFO mỗi tin, worker xử lý tách riêng.

Telethon có thể tự gửi RPC nội bộ để đồng bộ **trạng thái cập nhật** sau mất mạng; bản 4 tầng cũng có vòng `GetChannelDifference` chủ động trên đủ 22 kênh đang bật (theo danh sách trong `.env`), **không** gọi GetHistory. Khi socket đứt, các tin Telegram phát lúc mất kết nối **không được đảm bảo** bot sẽ xử lý lại, bởi cấu hình live-only không quét bù lịch sử. Hàng đợi RAM cũng mất khi dừng chương trình.

## Cài đặt trên Windows

1. Dừng bot cũ (`Ctrl+C`), sao lưu thư mục cũ.
2. Giải nén bản này vào thư mục mới, giữ cấu trúc tệp.
3. Bản sửa này giữ nguyên `.env` có sẵn trong ZIP đầu vào (bao gồm cấu hình và thông tin đăng nhập cá nhân). **Không chia sẻ bản ZIP đã sửa công khai.** Sao lưu và tự chuyển session Telethon, `data/`, cùng profile trình duyệt từ máy cũ nếu cần; những thư mục đó không được đóng gói.
4. Chạy `setup.bat` nếu chưa cài thư viện. Chạy `run.bat` và hoàn tất đăng nhập Telegram nếu được yêu cầu.
5. Kiểm tra log có `MTProto TCP PUSH SOCKET`, `handlers registered before socket connect`, `connected=True`; theo dõi `[Backpressure]` và `[Telegram admission]` để đánh giá tình trạng quá tải.

Điều chỉnh bằng `.env`:

```env
INGEST_QUEUE_MAXSIZE=128
MESSAGE_WORKERS=6
OCR_MEDIA_QUEUE_MAXSIZE=12
DOMAIN_QUEUE_MAXSIZE=64
TELEGRAM_RAW_DIAGNOSTICS=false
TELEGRAM_INGRESS_AGE_DROP_ENABLED=true
```

Khi quá tải: bot **ưu tiên phản hồi nhanh**, không bảo đảm giữ toàn bộ tin. Để không mất tin, cần thiết kế hàng đợi bền vững và replay/reconciliation có kiểm soát, không thể vừa cấm lưu/replay vừa bảo đảm nhận đủ tin qua mọi lần mất mạng.

Kiểm tra offline: `python -m unittest discover -s tests -v`. Test này không cần kết nối Telegram.

## Dọn các hàm không còn dùng (07/11/2026)

Đã kiểm tra các lời gọi trực tiếp, `getattr`/tên hàm dưới dạng chuỗi, API re-export,
callback của Telethon, Playwright, class override và các file `.bat` trước khi dọn.
Chỉ loại bỏ các hàm không còn đường gọi trong bản bot này:

| Tệp | Hàm loại bỏ | Lý do |
|---|---|---|
| `main_script.py` | `process_telegram_message` | Bộ nhận kiểu cũ, không được đăng ký; nay dùng `MTProtoPushSocket` |
| `main_script.py` | `_ocr_is_image_message` | Không còn nơi gọi |
| `main_script.py` | `_ocr_has_known_code_input_link` | Không còn nơi gọi |
| `scheduling.py` | `freshest_first_enabled` | Hàm chế độ cũ, không còn nơi gọi |
| `tg_ingest.py` | `TelegramReceiver.on_push` | Wrapper async cũ; socket gọi `offer_nowait` trực tiếp |
| `disk_guard.py` | `ensure_disk_space` | Không còn nơi gọi; hoạt động dọn định kỳ khác được giữ nguyên |
| `sc88_captcha.py` | `recognize_sc88_captcha` | Module độc lập không được import hoặc chạy (xóa cả file) |

Các import chỉ phục vụ phần mã đã xóa cũng được dọn. **Không xóa** các phương thức
được gọi qua tên chuỗi (`getattr`), các callback khi nhận sự kiện, method override,
hoặc chức năng bảo vệ submit, tải media, OCR, queue, khóa tài khoản và lưu DB.

Dọn mã chết chủ yếu giúp dự án dễ bảo trì; không tự làm tốc độ MTProto/socket
nhanh hơn. Để kiểm thử trên máy của bạn, chạy `python -m unittest discover -s tests -v`.

## Bản 4 tầng MTProto — Prewarm / Push+Raw+Difference / Admission / Worker (07/11/2026)

Bot được cập nhật trên bản MTProto đã dọn hàm thừa, **không đổi** HTTP submit,
Playwright, khóa tài khoản, trình tải/OCR, cấu hình website hoặc lịch sử kết quả.

```text
Telegram MTProto TCP — Telethon
  ├─ events.NewMessage / MessageEdited (không await công việc khác)
  ├─ events.Raw(UpdateNewChannelMessage / UpdateEditChannelMessage)
  └─ Active GetChannelDifference [22 kênh trong TELEGRAM_DIFF_CHAT_IDS, giới hạn RPC]
       ▲ InputPeerChannel(access_hash) + pts làm nóng từ get_dialogs(500)
       │ chỉ gọi GetFullChannel khi thiếu PTS; tối đa 120s startup
       ▼
 Lọc enabled, giờ mở cửa, chống trùng TTL 600s, tuổi 40/45/120s
       ▼
 Bounded RAM ingress; spoiler → text → image → video
       │ trong cùng loại, priority thấp hơn được phục vụ trước
       ├─ 6 message workers (16 tác vụ xử lý đồng thời toàn cục)
       ├─ Fast text/spoiler → trích mã
       └─ image/video → OCR queue riêng → giới hạn worker
       ▼
 Bounded domain queue → khóa tài khoản/tab → HTTP/Playwright submit
```

### Cấu hình `.env`

```dotenv
TELEGRAM_SYNC_DIALOGS=true
TELEGRAM_DIALOG_SYNC_LIMIT=500
TELEGRAM_DIALOG_SYNC_TIMEOUT=120.0
TELEGRAM_CHANNEL_TIMEOUT=10.0  # luồng Difference tự kẹp timeout thực tế tối đa 4s
TELEGRAM_ACTIVE_CHANNEL_DIFFERENCE=true
TELEGRAM_DIFF_PRIORITY_MAX=45
TELEGRAM_DIFF_INTERVAL_SEC=2.0
TELEGRAM_DIFF_CONCURRENCY=2
TELEGRAM_INGRESS_DEDUPE_TTL_SEC=600.0
INGRESS_XX88_MAX_AGE_SEC=40.0
INGRESS_OCR_MAX_AGE_SEC=45.0
INGRESS_MAX_AGE_SEC=120.0
INGEST_QUEUE_MAXSIZE=128
MESSAGE_WORKERS=6
MAX_CONCURRENT_PROCESSING=16
```

`TELEGRAM_DIFF_CHAT_IDS` cho phép chọn chính xác các kênh đồng bộ dự phòng;
mặc định bao phủ các kênh trong `CHANNEL_CONFIG`. Bản cấu hình hiện tại
chọn đủ 22 kênh, gồm cả RR88 và GG88 trước đây chỉ nhận Push/Raw.
Giới hạn tối đa cấu hình là 32 kênh,
trong `.env` hiện đặt `TELEGRAM_DIFF_MAX_CHANNELS=22`.
Khoảng cách RPC toàn cục tối thiểu 0,5 giây. Chu kỳ mỗi kênh tối thiểu 2 giây;
`answer.timeout` chỉ có thể đẩy lần quét kế tiếp tối đa tới 3 giây.
Giới hạn RPC dùng chung khiến một vòng 22 kênh cần ít nhất khoảng 11 giây;
đây là giới hạn đối soát dự phòng, không phải độ trễ của Push.
**Cấu hình này tăng đáng kể số RPC, có thể dẫn đến FloodWait của Telegram.**
Nếu phát sinh FloodWait, giảm danh sách `TELEGRAM_DIFF_CHAT_IDS` hoặc tăng
`TELEGRAM_DIFF_INTERVAL_SEC` (chẳng hạn 6–15 giây). FloodWait tạm dừng
RPC dự phòng của tất cả các kênh, không chặn handler Push/Raw. Có thể tắt bằng
`TELEGRAM_ACTIVE_CHANNEL_DIFFERENCE=false` để trở về Push/Raw-only.

Nếu mất kết nối, Telethon tự phục hồi state; không gọi `GetHistory` và không
bắt buộc chạy `catch_up` tại startup. `GetChannelDifference` chỉ cố gắng đối
chiếu tin gần thời gian chạy từ PTS hiện có, không phải cơ chế đảm bảo thu đủ
mọi tin. Timeout RPC tối đa 4s, concurrency 2, interval tối thiểu 1,5s,
backoff khi lỗi/FloodWait, tối đa 3 trang mỗi đợt. Đặc biệt tránh chạy nhiều
bản bot cùng một session Telegram vì dễ trùng RPC/khóa SQLite.

Không thể cam kết cửa lọc luôn dưới 1ms hay loại bỏ hoàn toàn độ trễ push nếu
không benchmark mạng/DC thật và máy Windows. Dưới tải quá lớn, RAM queue có
thể chủ động bỏ tin ưu tiên thấp, và 600s TTL được giới hạn 10.000 khóa.

Không bảo đảm chống trùng vĩnh viễn: cache nhận tin giữ tối đa 10.000 phiên bản
trong 600 giây; tin sửa có nội dung mới vẫn được xử lý. Các lớp bảo vệ gửi lại
kết quả chưa rõ vẫn giữ nguyên. Dữ liệu tin không ghi vào SQLite inbox.
DB kết quả được dọn nền mỗi 300 giây: giữ 30 ngày, tối đa 20.000 dòng mỗi bảng
chi tiết và 100.000 dòng mỗi bảng guard; file WAL được checkpoint định kỳ.
Lịch sử CSV/JSONL quá 30 ngày được dọn cùng tác vụ nền; marker kết quả chưa rõ
trong RAM chỉ giữ ngày hiện tại. Vì vậy dữ liệu mới không cần là tin trùng
thì vẫn được kiểm soát kích thước.

**Tự kiểm tra:** `python -m unittest discover -s tests -v`; kiểm tra log
`[MTProtoPrewarm]`, `[MTProtoDiff]`, `[Telegram admission]`, `[Backpressure]`.

## Bản vá đối chiếu lỗi 07/11/2026

- `config.py`: các hằng/hàm Dual Spoiler/OCR **đã có sẵn**, không khai báo lại;
  cập nhật danh sách 11 kênh fallback và giới hạn luồng; thay thứ tự khung hình
  XX88 thành `7,8,9,10,11`.
- `browser_engine/cdp.py`: bắt thêm `AttributeError` nếu phiên bản launcher
  khác chưa cung cấp hàm kiểm tra profile. `chrome_launcher.active_profile_info`
  **đã tồn tại** trong bản ZIP gốc.
- `mtproto_push.py`: chỉ nâng PTS khi `event.pts == state.pts + event.pts_count`;
  tuyệt đối không nhảy qua khoảng trống chưa được GetChannelDifference xử lý.
- `mtproto_channel_diff.py`: nới danh sách kênh và khoảng thời gian cấu hình;
  vẫn giữ backoff khi FloodWait. Phải theo dõi số RPC/flood thực tế.
- `tg_ingest.py`: KJC dùng tuổi tin tổng quát 120s, không bị áp nhầm 40s của XX88.
- `main_script.py`: mã spoiler phải giải được và vượt kiểm tra hợp lệ mới được
  nhập; text được kiểm tra định dạng theo domain; OCR XX8 vẫn bắt buộc mã ASCII
  10 ký tự bắt đầu đúng `XX` và phân biệt chữ hoa/thường.
- `code_validator.py`: không tự động coi mọi chuỗi chứa `KT` là placeholder;
  chỉ bỏ qua placeholder có dấu chưa giải quyết. Giảm log INFO trên fast-path.
- `.env`: cấu hình fallback được cập nhật đồng bộ với mã, bảo tồn các giá trị
  khác trong bản ZIP đầu vào. Không thay đổi cơ chế gửi mã, HTTP/CDP, OCR hay
  các thông số tài khoản ngoài phạm vi các sửa lỗi trên.

**Kiểm tra offline:** 68 file Python qua kiểm tra cú pháp; 33 kiểm tra giả lập
cấu hình, giải mã/loại bỏ mã và PTS/khôi phục tin. Chưa kiểm tra đăng nhập
Telegram, website, CAPTCHA hoặc submit thực tế trên Windows.
Môi trường offline không có Telethon; kiểm tra bridge dùng mô phỏng module.
Không có file test, log, cache bytecode hoặc file `.bak` trong ZIP phân phối.


## Bản vá bổ sung — Username / khôi phục tab / fanout stateless (07/11/2026)

- `browser_engine/js_scripts.py`: so khớp tên tài khoản **không phân biệt chữ hoa/thường** trong `PROFILE_FILL_JS`, `_TAB_ONLY_FILL_JS`, `_PREFILL_TAB_USERNAME_JS`. Khi tài khoản đã điền sẵn cùng giá trị theo cách so khớp này, không viết lại chỉ để đổi kiểu chữ.
- `browser_engine/forms.py` và `browser_engine/submit_inner.py`: đồng bộ đối chiếu username Python bằng `casefold()` (vẫn giữ kiểm tra chính xác nội dung mã code theo từng site), kể cả kiểm tra trước khi submit.
- `browser_engine/page_recovery.py`: danh sách khôi phục tab-only dùng `BROWSER_DOMAINS` đã bật, thay vì chỉ `llwincode.com`. Giữ guard Cloudflare/slider/xác minh và cooldown 20 giây mỗi tab. `submit_inner.py` còn thử điền lại khi SPA khôi phục form hoặc mất ô code; không chuyển sang tài khoản khác khi tên tài khoản đang hiện không khớp.
- `browser_engine/__init__.py`: re-export `_DISMISS_XX88_RESULT_POPUP_JS` từ `js_scripts.py`.
- `main_script.py`: `STATELESS_ENTRY_ACCOUNTS=all` nhân từng mã cho mỗi **tab browser đang hoạt động** (tối đa số luồng domain). `one` và HTTP vẫn chỉ tạo một item/mã. Mỗi item `all` mang `tab_slot`; các lớp `submit_code_with_delay`, `submit_code_safe`, `browser_engine.submit_code_browser_tab`, `TabPool.acquire` truyền/giữ `tab_slot` tới đúng tab thay vì lựa tab rảnh tùy ý. Chống trùng và in-flight theo `(domain,tab_slot,code)` để bản sao thứ hai không bị chặn nhầm.
- Nếu tab được đóng/sắp xếp lại khi đang xử lý, item gắn slot cũ có thể phải hoãn hoặc bỏ. Không tự ý gửi vào tab khác. Kiểu chữ username có thể khác nhau, nhưng mã code không bị ép lowercase.

**Kiểm tra offline:** 68/68 tệp Python phân tích cú pháp thành công; 5 ca mô phỏng DOM JavaScript, 6 ca fanout và 3 ca chọn tab đều đạt. Chưa kiểm thử với phiên Telegram/CDP thật trên Windows. Server có thể từ chối mã đã sử dụng khi gửi lại sang tài khoản/tab khác. Không tự vượt Cloudflare/captcha.

**Bảo mật:** dự án ZIP giữ `.env` từ bản của người dùng. Không chia sẻ công khai ZIP hoặc thông tin API/session bên trong. `setup.bat` cài môi trường riêng nếu cấu hình hiện tại còn phù hợp với máy.
