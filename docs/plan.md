# Kế hoạch: Plugin KOReader đọc sách từ Google Drive (offline-first)

> Cập nhật 2026-10-25 theo phản hồi: thiết bị Kobo; đã có OAuth client id/secret; thư viện = nhiều folder nguồn do user thêm (gồm cả folder được chia sẻ); hạn mức dung lượng do user cấu hình; **không** đồng bộ ngược tiến độ đọc (để dành cho kế hoạch sau).
>
> Cập nhật lần 2: dùng chung OAuth client với rclone (Desktop); nguồn lớn nhất ~200–300 GB, nhiều loại file, cả truyện tranh lẫn truyện chữ; thông số đọc trực tiếp từ Kobo qua USB (mục 5.6).

## 1. Bối cảnh & mục tiêu

**Mục tiêu:** KOReader trên máy đọc sách duyệt được các thư mục sách trên Google Drive, tải từng cuốn về khi cần đọc, và vẫn duyệt/đọc được khi Wi-Fi tắt.

**Ràng buộc:**

| Ràng buộc | Hệ quả thiết kế |
|---|---|
| Bộ nhớ thiết bị nhỏ | Chỉ tải sách khi cần, hạn mức dung lượng do user đặt, tự xoá bớt (LRU) |
| CPU/RAM yếu, không duyệt Drive trực tiếp được | Chỉ gọi REST API tối giản, gộp request, UI dạng danh sách văn bản, phân trang |
| Wi-Fi thường tắt | **Offline-first**: mọi thao tác duyệt chạy trên catalog cache cục bộ; mạng chỉ dùng khi user chủ động "Sync" |
| PDF chỉ đọc tốt bằng KOReader | Giải pháp nằm *bên trong* KOReader (plugin Lua), mở sách bằng reader của KOReader |

**Luồng sử dụng:**

1. User bật Wi-Fi → **Sync**: cập nhật catalog của mọi folder nguồn + tải các sách trong hàng đợi.
2. User tắt Wi-Fi → vẫn duyệt toàn bộ cây thư mục từ cache, thấy cuốn nào đã có trên máy, cuốn nào chưa.
3. Chọn cuốn đã tải → mở ngay. Chọn cuốn chưa tải → có mạng thì tải rồi mở; không có mạng thì đưa vào **hàng đợi**, lần sync sau sẽ tải.

**Ngoài phạm vi (để sau):** đồng bộ tiến độ đọc/highlight giữa nhiều thiết bị, backup lên Drive. Thiết kế hiện tại chỉ cần scope **chỉ đọc**; khi làm phần này sẽ cần xin thêm scope ghi.

---

## 2. Các phương án kiến trúc

### Phương án A: plugin KOReader gọi thẳng Google Drive API v3 (**đã chọn**)

- Plugin Lua (`gdrive.koplugin`) dùng `ssl.https`/`socket.http` có sẵn trong KOReader.
- Không cần máy chủ trung gian, chạy độc lập hoàn toàn trên Kobo.

### Phương án B: rclone làm cầu nối + WebDAV/OPDS có sẵn của KOReader

- Trên PC/NAS chạy `rclone serve webdav gdrive:Books`; KOReader dùng **Cloud storage → WebDAV** có sẵn.
- Nhược điểm: cần máy luôn bật, không có catalog offline, hàng đợi hay quản lý dung lượng.
- Chỉ dùng làm **giải pháp tạm** (Phase 0) nếu cần đọc ngay.

---

## 3. Xác thực Google

### 3.1 Ràng buộc từ phía Google

- **Scope cần dùng:** `https://www.googleapis.com/auth/drive.readonly`. Scope này đọc được mọi file/folder user có quyền xem: My Drive, *Shared with me*, Shared drives.
- **Device flow (nhập mã trên điện thoại) không dùng được:** với device flow, Google chỉ cho scope `drive.file`/`drive.appdata`, tức là chỉ thấy file do app tạo ra, không duyệt được thư viện có sẵn.
- **Loại OAuth client:** nên là **Desktop app**, vì loại này cho phép redirect về loopback `http://127.0.0.1:<port>` mà không phải đăng ký trước. Nếu client hiện có là *Web application* thì phải thêm redirect URI `http://127.0.0.1:8765/` vào Console, hoặc tạo thêm một client Desktop (miễn phí, tạo nhanh).
- **Consent screen phải ở trạng thái "In production".** Nếu để *Testing*, refresh token **hết hạn sau 7 ngày**. App chưa verify vẫn dùng được cho cá nhân (giới hạn 100 user, lúc đăng nhập hiện cảnh báo "Google hasn't verified this app" → *Advanced → Go to …*).
- Luôn dùng **PKCE** + tham số `access_type=offline&prompt=consent` để chắc chắn nhận được refresh token.

### 3.2 Các cách lấy refresh token (gợi ý)

| Cách | Mô tả | Đánh giá |
|---|---|---|
| **A. Script trên PC** | `tools/gdrive_login.py` (Python thuần, không cần thư viện ngoài): mở trình duyệt → đăng nhập → nhận code qua loopback → đổi lấy refresh token → ghi file `gdrive_auth.lua`. Chép vào Kobo qua USB. | Chắc chắn chạy, làm được ngay. **Dùng cho MVP.** |
| **B. Ghép đôi qua điện thoại trong mạng LAN** (không cần PC) | Plugin mở một HTTP server nhỏ trên Kobo và hiện **QR code** trỏ tới `http://<ip-kobo>:8765/`. Điện thoại quét QR → trang trên Kobo có nút "Đăng nhập Google". Sau khi đồng ý, Google redirect điện thoại tới `http://127.0.0.1:8765/?code=…`, trang này *lỗi không tải được* (đó là chuyện bình thường). User **copy URL trên thanh địa chỉ**, quay lại tab trang Kobo, dán vào ô và bấm gửi. Kobo tách lấy `code` và tự đổi lấy refresh token. | Không cần PC, không phải gõ trên bàn phím e-ink; hơi nhiều bước nhưng chỉ làm một lần. **Làm ở Phase 5.** |
| C. Service account | Chia sẻ folder cho email của service account; Kobo tự ký JWT RS256. | Phải ký RSA trong Lua qua FFI OpenSSL → phức tạp; folder phải share riêng cho nó. Chỉ là dự phòng. |
| D. API key + folder "Anyone with link" | Không cần OAuth. | Sách bị public theo link. Chỉ dùng nếu chấp nhận điều đó. |

**Đề xuất:** A cho MVP, sau đó B để thiết bị tự đăng nhập lại khi token bị thu hồi.

### 3.2b Dùng chung client với rclone (đã chốt)

Client hiện có là loại **Desktop**, đang dùng cho rclone → dùng lại được, và **rclone thay luôn được script đăng nhập ở cách A**:

```powershell
# Tạo một remote riêng chỉ đọc cho Kobo. rclone tự mở trình duyệt để đăng nhập.
rclone config create kobo-gdrive drive client_id=<ID> client_secret=<SECRET> scope=drive.readonly
# Xem token vừa tạo (trong đó có refresh_token)
rclone config dump
```

Sau đó `tools/rclone_to_kobo.py` đọc `rclone config dump` và ghi ra `gdrive_auth.lua` (client_id, client_secret, refresh_token), có thể ghi thẳng vào `E:\.adds\koreader\settings\` khi Kobo đang cắm USB.

Lưu ý:

- **Tạo token riêng với scope `drive.readonly`**, đừng lấy token của remote rclone đang dùng (thường có scope `drive` toàn quyền). Hai token độc lập nhau.
- Nếu remote rclone hiện tại đã chạy được hơn 7 ngày mà không phải đăng nhập lại thì consent screen đã ở **In production**, không cần chỉnh gì thêm.
- **Thu hồi:** gỡ app trong Google Account → *Third-party access* sẽ thu hồi **mọi** token của client này, kể cả rclone. Muốn thu hồi riêng token của Kobo thì gọi `https://oauth2.googleapis.com/revoke?token=<refresh_token>` (plugin sẽ có nút "Đăng xuất" làm việc này).
- Quota Drive API tính theo project, dùng chung với rclone, nhưng hạn mức mặc định rất rộng so với nhu cầu này.

### 3.3 Trên thiết bị

- File `koreader/settings/gdrive_auth.lua`: `client_id`, `client_secret`, `refresh_token`.
- Khi có mạng: `POST https://oauth2.googleapis.com/token` (grant_type=refresh_token) → access token (~1 giờ), giữ trong RAM; gặp 401 thì refresh; gặp `invalid_grant` thì báo "Cần đăng nhập lại" kèm hướng dẫn.
- Refresh token nằm dạng plaintext trên Kobo. Chấp nhận được vì scope chỉ đọc; README sẽ hướng dẫn thu hồi ở myaccount.google.com → Security → Third-party access nếu mất máy.
- **Đồng hồ Kobo:** nếu giờ hệ thống sai nhiều thì TLS và token đều lỗi. Plugin kiểm tra header `Date` của response, cảnh báo khi lệch lớn.

---

## 4. Thư viện: folder nguồn & cách duyệt cây thư mục

### 4.1 Folder nguồn (source)

User quản lý danh sách nguồn trong *Google Drive → Nguồn sách*. Mỗi nguồn có:

- `folder_id`, tên hiển thị (đổi được), loại (`my_drive` / `shared_with_me` / `shared_drive`), `drive_id` nếu là shared drive.
- **Chế độ đồng bộ:**
  - `full`: tải metadata của toàn bộ cây (mặc định cho folder vừa phải);
  - `depth N`: chỉ N cấp, folder sâu hơn để trạng thái "chưa đồng bộ";
  - `on_demand`: chỉ đồng bộ một folder khi user mở nó lúc đang online (dành cho folder khổng lồ).
- Bật/tắt nguồn (tạm ẩn mà không mất dữ liệu đã tải).

**Cách thêm nguồn** (có nhiều cách vì gõ trên Kobo rất cực):

1. **Folder picker online ngay trên Kobo** (cách chính): duyệt lười từng cấp, mỗi lần chỉ một request:
   - *My Drive* → `'root' in parents`
   - *Shared with me* → `sharedWithMe = true and mimeType = 'application/vnd.google-apps.folder'`
   - *Shared drives* → `drives.list`
   - *Starred* → `starred = true` (lối tắt tiện cho folder hay dùng)

   Giữ lâu một folder → "Thêm làm nguồn".
2. **Dán link/ID qua trang web LAN** (cùng HTTP server với cách đăng nhập B): mở trang trên điện thoại/PC, dán link `drive.google.com/drive/folders/<id>`.
3. **Sửa file cấu hình** `gdrive_sources.lua` qua USB (cho người thích tự làm).

### 4.2 Các cách đồng bộ cây thư mục (brainstorm)

Chi phí chính trên Kobo là **số lượng request HTTPS** (mỗi request ~0,3–1 giây kể cả bắt tay TLS trên Wi-Fi yếu), không phải dung lượng dữ liệu. Các cách:

| # | Cách | Số request | Nhận xét |
|---|---|---|---|
| 1 | **BFS ngây thơ**: mỗi folder một lệnh `files.list` | = số folder | 500 folder ≈ 5–8 phút. Quá chậm. |
| 2 | **BFS gộp cha**: `q = ('id1' in parents or 'id2' in parents or … ) and trashed = false`, gộp 20–40 folder mỗi request (giới hạn bởi độ dài URL), `pageSize=1000` | ≈ số folder / 30 + số trang | Giảm 20–40 lần. Làm được cho mọi loại nguồn. **Mặc định.** |
| 3 | **Quét cả shared drive một lần**: `corpora=drive&driveId=…&includeItemsFromAllDrives=true&supportsAllDrives=true`, liệt kê mọi item rồi dựng cây cục bộ theo `parents` | ≈ số item / 1000 | Nhanh nhất với shared drive chủ yếu chứa sách. Nếu drive có nhiều file rác (ảnh…) thì lọc theo `mimeType`/đuôi file trong `q`. |
| 4 | **Lấy hết folder trước, file sau**: một truy vấn `mimeType = folder` cho toàn bộ My Drive → dựng cây → chỉ lấy file của các folder nằm trong nguồn (dùng cách 2) | ít | Tốt khi My Drive có nhiều folder nhưng nguồn chỉ là một nhánh nhỏ. Tối ưu thêm, không bắt buộc. |
| 5 | **Tăng dần bằng Changes API**: `changes.getStartPageToken` sau lần quét đầy đủ, các lần sau `changes.list` (riêng cho My Drive/Shared-with-me và cho từng shared drive) | thường 1–2 | Sync hằng ngày gần như tức thì. Mỗi thay đổi được đối chiếu với cây đã biết: item có cha nằm trong nguồn thì cập nhật, item bị chuyển ra ngoài hoặc xoá thì gỡ. |
| 6 | **Lười theo folder** (`on_demand`): mở folder khi online thì gọi `files.list` cho đúng folder đó và cache lại | 1 mỗi lần mở | Cho folder khổng lồ; offline chỉ thấy những gì đã từng mở. |

**Lọc ngay trong truy vấn (quan trọng với nguồn 200–300 GB nhiều loại file):** thêm vào `q` các điều kiện `not mimeType contains 'video/' and not mimeType contains 'audio/'`; ảnh (`image/`) cũng bị loại, trừ khi nguồn bật chế độ truyện tranh dạng folder ảnh (4.4). Drive không lọc được theo đuôi file trong `q`, nên phần còn lại lọc theo đuôi ngay trên thiết bị, và **chỉ lưu vào SQLite những item là sách hoặc folder chứa sách**. Nhờ vậy catalog nhỏ và lượng dữ liệu tải về giảm mạnh.

**Ước lượng cho nguồn lớn nhất:** giả sử khoảng 30.000 file sách và 3.000 folder:

- Quét lần đầu mất khoảng 100–150 request, cỡ 2–5 phút. Chạy một lần, có tiến độ, bị ngắt thì chạy tiếp được.
- Catalog khoảng 5–15 MB.
- Các lần sync sau qua Changes API: vài giây.

Nếu thực tế lớn hơn nhiều thì đặt nguồn này ở chế độ `depth 2` hoặc `on_demand`.

**Chiến lược đề xuất:** lần đầu quét đầy đủ bằng **cách 2** (hoặc **cách 3** cho shared drive); các lần sau dùng **cách 5**; folder nguồn đặt `on_demand` thì dùng **cách 6**. Có thêm nút "Quét lại toàn bộ" để sửa khi cache lệch.

> Lưu ý: không dựa vào `modifiedTime` của folder để phát hiện thay đổi, vì Drive không cập nhật trường này một cách tin cậy khi file con thay đổi. Changes API mới là nguồn đáng tin.

**Các trường hợp đặc biệt của Drive:**

- **Shortcut** (`application/vnd.google-apps.shortcut`): lấy `shortcutDetails(targetId, targetMimeType)`; shortcut trỏ tới folder thì duyệt tiếp vào target, có **chống vòng lặp** (tập folder đã thăm).
- **Một file nằm ở nhiều nơi** (nhiều nguồn chồng nhau, shortcut): lưu quan hệ cha–con nhiều-nhiều; file tải về **một lần** theo `file_id`, các chỗ khác cùng trỏ tới.
- **Nguồn lồng nhau** (nguồn A chứa nguồn B): không quét trùng, B dùng lại dữ liệu của A.
- **Mất quyền truy cập** (bị bỏ chia sẻ, 404/403): đánh dấu nguồn "không truy cập được", giữ sách đã tải, hỏi user có xoá không.
- **Google Docs/Sheets native, ảnh, video…**: bỏ qua. Chỉ giữ các định dạng KOReader đọc được (pdf, epub, djvu, cbz, cbr, mobi, azw3, fb2, txt, html, rtf, doc(x)…); danh sách sửa được trong Settings.
- **Folder rỗng sau khi lọc**: tuỳ chọn ẩn (mặc định ẩn) để cây gọn.

**Trường dữ liệu yêu cầu từ API** (giữ response nhỏ):
`nextPageToken, files(id, name, mimeType, size, modifiedTime, md5Checksum, parents, shortcutDetails, trashed)`

### 4.3 Cách duyệt offline (UI)

Màn hình gốc của plugin:

```
Google Drive                      Sync: 2 ngày trước
───────────────────────────────────────────────
🔍 Tìm kiếm
🕘 Đọc gần đây
🆕 Mới trên Drive (từ lần sync trước)   (7)
💾 Trên máy                       1,2/2 GB
⏳ Hàng đợi tải                         (3)
───────────────────────────────────────────────
📚 Sách giáo khoa          (124 · 18 đã tải)
📚 Truyện (shared)          (56 · 4 đã tải)
📚 Papers [on-demand]
```

- **Cây thư mục theo nguồn:** giống hệt trên Drive. Mỗi folder hiện số lượng `(sách · đã tải)`, tính sẵn khi sync để mở folder không phải đếm lại.
- **Các danh sách phẳng** (giải quyết việc cây quá sâu, đi tìm mệt trên e-ink):
  - *Tìm kiếm* theo tên trên mọi nguồn (SQL `LIKE`, chạy offline);
  - *Đọc gần đây*; *Mới trên Drive* (thêm/sửa từ lần sync trước); *Trên máy*; *Hàng đợi*.
- **Rút gọn chuỗi folder chỉ có một con:** `A / B / C` hiện thành một dòng (tuỳ chọn).
- **Biểu tượng trạng thái mỗi dòng:** `✓` đã có trên máy · `☁` chưa tải · `⏳` đang chờ · `↻` trên Drive có bản mới hơn · `📌` đã ghim · `⚠` lỗi tải. Kèm kích thước file.
- **Chạm:** folder → mở; sách trên máy → mở bằng `ReaderUI:showReader(path)`; sách chưa tải → tải (nếu online) hoặc thêm vào hàng đợi.
- **Giữ lâu:** Tải / Xếp hàng / Huỷ / Ghim / Xoá bản trên máy / Thông tin; với folder: *Xếp hàng cả folder* (có hỏi lại, hiện tổng dung lượng).
- **Sắp xếp:** tên (sắp xếp tự nhiên, "Tập 2" đứng trước "Tập 10") / ngày sửa / kích thước. Cảnh báo khi file > ngưỡng (mặc định 100 MB).
- Dùng `Menu` widget có sẵn (danh sách văn bản, phân trang theo màn hình); không có thumbnail ở v1.
- Folder `on_demand` chưa từng mở mà đang offline: hiện "Chưa đồng bộ – bật Wi-Fi để xem".

### 4.4 Truyện tranh & nhiều loại file

Nguồn có cả truyện chữ lẫn truyện tranh, nên phải xử lý các dạng sau:

| Dạng trên Drive | Cách xử lý |
|---|---|
| pdf, epub, mobi, azw3, fb2, djvu, txt… | Tải nguyên file, mở bằng KOReader |
| **cbz / zip** | Tải nguyên file; KOReader mở trực tiếp |
| **cbr / rar, cb7 / 7z** | Tải nguyên file. KOReader v2025.04 trên máy có `libarchive` (plugin Archive viewer), nhưng **phải thử mở một file .cbr/.cb7 thật ở Phase 1**. Nếu không đọc được như một cuốn truyện thì cho hiện kèm cảnh báo, hoặc (tuỳ chọn) chuyển thành .cbz ngay trên máy sau khi tải bằng `ffi/archiver` (giải nén) + `ffi/zipwriter` (đóng gói) |
| **Folder chứa toàn ảnh** (mỗi chapter là một folder jpg/png/webp) | Bật chế độ **"Folder ảnh = truyện"** cho từng nguồn (mặc định tắt). Khi đó folder có ≥ N ảnh và không có folder con hiện như **một cuốn sách** (`📖 Chap 12 · 45 trang · 18 MB`). Lúc tải: lấy lần lượt từng ảnh rồi đóng gói thành **.cbz** trên máy bằng `ffi/zipwriter` (chế độ store, không nén lại). Cần nhiều request nên chậm hơn, nhưng chạy trong hàng đợi và resume được theo từng ảnh |
| Ảnh lẻ, video, audio, Google Docs native, file khác | Bỏ qua |

Phần duyệt bổ sung cho truyện tranh:

- Sắp xếp tự nhiên (`Chap 2` trước `Chap 10`), quan trọng với tên chapter.
- Giữ lâu một folder bộ truyện → **"Tải N chương tiếp theo"**: tính từ chương đang đọc dở hoặc chương đầu chưa đọc; số N chỉnh được.
- Tuỳ chọn: tự xoá chương đã đọc xong khi cần chỗ (ưu tiên xoá trước LRU thường).

**Cách đã xem xét và loại bỏ:** tạo file giả 0 byte trong File Manager của KOReader để duyệt bằng giao diện có sẵn. Loại vì KOReader sẽ cố mở hoặc đọc metadata các file đó, làm rối History/Collections, và không có chỗ hiển thị trạng thái. Thay vào đó, sách **đã tải** vẫn nằm trong thư mục thật nên File Manager, History, Collections của KOReader dùng bình thường.

---

## 5. Thiết kế plugin

### 5.1 Cấu trúc thư mục

```
gdrive.koplugin/
├── _meta.lua          -- tên, mô tả plugin
├── main.lua           -- đăng ký menu, Dispatcher actions, điều phối
├── auth.lua           -- đọc gdrive_auth.lua, refresh access token
├── api.lua            -- Drive API: files.list (gộp cha), drives.list, changes, download
├── catalog.lua        -- SQLite: sources, items, edges, queue
├── sources.lua        -- quản lý nguồn + folder picker online
├── sync.lua           -- quét đầy đủ / tăng dần / on-demand, xử lý hàng đợi
├── downloader.lua     -- tải ra .part, Range resume, kiểm tra size, tiến độ
├── storage.lua        -- hạn mức, LRU eviction, ghim
├── browser.lua        -- UI duyệt offline (Menu widget), tìm kiếm, danh sách phẳng
└── pairing.lua        -- (Phase 5) HTTP server LAN: đăng nhập qua điện thoại, thêm nguồn
tools/
└── gdrive_login.py    -- script trên PC sinh gdrive_auth.lua
```

### 5.2 Lược đồ SQLite

File: `koreader/settings/gdrive/catalog.sqlite3` (KOReader có sẵn `lua-ljsqlite3`).

```sql
sources(id PK, folder_id, name, kind, drive_id, sync_mode, depth,
        enabled, status, changes_token, last_sync)
items(id PK,            -- Drive file id
      name, mime_type, is_folder, size, modified_time, md5,
      shortcut_target,
      book_count, local_count,           -- chỉ cho folder, tính sẵn khi sync
      synced,                            -- folder on_demand/depth: đã liệt kê con chưa
      local_path, local_modified_time,
      state,            -- remote|queued|downloading|local|outdated|error
      pinned, last_opened, first_seen)
edges(parent_id, child_id, PRIMARY KEY(parent_id, child_id))  -- nhiều-nhiều
meta(key PK, value)     -- last_sync, schema_version, ...
```

- Index: `edges(parent_id)`, `items(name COLLATE NOCASE)`, `items(state)`, `items(last_opened)`.
- Mọi lần ghi khi sync gói trong **một transaction**. Tính lại `book_count/local_count` theo từng nhánh sau khi sync.

### 5.3 Tải sách

- `GET /drive/v3/files/{id}?alt=media&supportsAllDrives=true`, ghi thẳng ra file bằng `ltn12.sink.file` (không giữ file trong RAM, quan trọng với PDF lớn).
- Ghi ra `*.part`, xong mới rename. Wi-Fi rớt thì resume bằng `Range: bytes=<size .part>-`.
- Kiểm tra `size` khớp metadata; MD5 để tuỳ chọn (chậm trên máy yếu với file lớn).
- Trước khi tải: kiểm tra dung lượng trống thật trên ổ và hạn mức → chạy eviction hoặc hỏi user.
- Tiến độ + nút huỷ; hàng đợi chạy tuần tự (một file một lúc).
- Thư mục lưu: `<home>/.gdrive/<tên nguồn>/<đường dẫn như trên Drive>/` (xem 5.6 về lý do dùng thư mục bắt đầu bằng dấu chấm trên Kobo). File có nhiều đường dẫn thì lưu theo đường dẫn đầu tiên.
- Khi file trên Drive đổi tên/chuyển chỗ: đổi tên/chuyển file trên máy cho khớp. Nếu KOReader đang dùng docsettings theo hash thì tiến độ đọc không mất.

### 5.4 Quản lý dung lượng (hạn mức do user đặt)

- Settings: **hạn mức** (ví dụ 500 MB / 1 GB / 2 GB / tuỳ chọn), **dung lượng trống tối thiểu phải chừa lại trên ổ**, **ngưỡng cảnh báo file lớn**.
- Vượt hạn mức → xoá theo **LRU** (`last_opened` cũ nhất), bỏ qua sách đã ghim và sách đang mở; trước khi xoá hiện danh sách sẽ bị xoá để user xác nhận (có tuỳ chọn "tự động, không hỏi").
- Sách bị xoá trở về trạng thái `remote`, vẫn hiện trong catalog và tải lại được.
- **Giữ tiến độ đọc khi xoá sách:** máy đang để `document_metadata_folder = "doc"`, tức là `.sdr` nằm cạnh file sách, nên xoá sách là mất tiến độ. Cần đổi *Settings → Document → Book metadata location* sang **"hash"** (hoặc "koreader/docsettings"). Khi đổi, KOReader sẽ hỏi có chuyển metadata hiện có sang chỗ mới không: chọn **có**. Khi khởi động, plugin cảnh báo nếu còn để "doc".
- Màn hình *Trên máy*: liệt kê sách, dung lượng, sắp xếp theo lần đọc gần nhất, xoá/ghim hàng loạt.

### 5.5 Wi-Fi

- `NetworkMgr:isOnline()` để quyết định tải ngay hay xếp hàng.
- Bấm Sync: `NetworkMgr:runWhenOnline(cb)` (KOReader sẽ hỏi bật Wi-Fi).
- Tuỳ chọn: **tự tắt Wi-Fi sau khi sync + tải xong** (tiết kiệm pin); **tự sync khi Wi-Fi vừa kết nối** (event `NetworkConnected`, mặc định tắt).
- Timeout ngắn cho từng request, retry có backoff, xử lý 429/5xx; giữa các trang kết quả gọi `UIManager:scheduleIn` để UI không bị treo; sync bị ngắt giữa chừng thì lần sau chạy tiếp (trạng thái nằm trong SQLite).

### 5.6 Lưu ý riêng cho Kobo

**Thông số đọc từ máy (ổ E:, 2026-10-25):**

| Mục | Giá trị | Hệ quả |
|---|---|---|
| KOReader | **v2025.04** (`.adds/koreader/git-rev`) | Phiên bản mục tiêu để phát triển và test |
| Firmware Kobo | 4.38.23429 | |
| Bộ nhớ | 6,9 GB, **chỉ còn trống 39 MB** | **Phải dọn chỗ trước khi test** (Phase 0.5) |
| Sách hiện có | `/mnt/onboard/Home` = 6,3 GB (home_dir của KOReader) | Ứng viên chuyển lên Drive |
| Dữ liệu KOReader | `ota/` 91 MB (gói cập nhật cũ), `cache/` 47 MB | Xoá được, lấy lại ~140 MB |
| Nickel | `KoboReader.sqlite` 139 MB; `ExcludeSyncFolders` đã bỏ qua thư mục bắt đầu bằng dấu chấm | Đặt thư mục tải là `.gdrive` thì Nickel không quét |
| Metadata sách | `document_metadata_folder = "doc"` | Đổi sang "hash" (5.4) |
| Thư viện có sẵn | `libsqlite3`, `libarchive`, `libssl/libcrypto`, `ffi/zipwriter.lua`, `ffi/archiver.lua` | Đủ cho catalog, đóng gói .cbz, HTTPS |
| Plugin có sẵn | `SSH.koplugin`, `httpinspector`, `qrclipboard` | SSH dùng để phát triển; QR dùng cho ghép đôi qua điện thoại |

**Các lưu ý khác:**

- **Nickel tự quét sách:** thư mục tải đặt là `/mnt/onboard/.gdrive/`. Thư mục `Home` hiện tại *không* có dấu chấm nên Nickel vẫn quét nó (một phần lý do `KoboReader.sqlite` lớn).
- Đặt `.gdrive` làm lối tắt trong File Manager của KOReader để dễ vào.
- Catalog SQLite cũng nằm trên `/mnt/onboard`, nên được tính vào hạn mức.
- Đồng hồ hệ thống (xem 3.3).

### 5.7 Điểm truy cập

- Menu chính: **Tools → Google Drive**: *Duyệt*, *Sync ngay*, *Hàng đợi*, *Trên máy*, *Nguồn sách*, *Cài đặt*, *Tài khoản*.
- Action cho **Gestures/Dispatcher**: "Mở Google Drive", "Sync Google Drive".

---

## 6. Lộ trình triển khai

| Phase | Nội dung | Kết quả |
|---|---|---|
| **0** (tuỳ chọn) | rclone `serve webdav` + Cloud storage WebDAV của KOReader | Đọc được ngay trong lúc chờ plugin |
| **0.5. Dọn chỗ** | Xoá `ota/` và `cache/` của KOReader; đổi metadata sang "hash"; sao lưu `Home` lên Drive rồi xoá bớt sách ít đọc | Còn trống tối thiểu ~500 MB để test |
| **1. Spike** | Tạo token bằng rclone + `rclone_to_kobo.py`; trên Kobo: refresh token, list 1 folder, tải 1 file, kiểm tra TLS và đồng hồ; mở thử .cbz/.cbr/.cb7 | Chứng minh OAuth và HTTPS chạy trên Kobo thật; biết định dạng truyện nào đọc được |
| **2. Nguồn + catalog** | SQLite schema, folder picker online (My Drive / Shared with me / Shared drives / Starred), quét đầy đủ kiểu BFS gộp cha, shortcut, lọc định dạng | Thêm được nhiều nguồn, có cache cây thư mục |
| **3. Duyệt offline + tải** | `browser.lua` (cây + Tìm kiếm + Trên máy + Hàng đợi), downloader (.part, resume), hàng đợi offline | **MVP**: luồng sử dụng chính chạy được |
| **4. Dung lượng** | Hạn mức, LRU, ghim, cảnh báo docsettings, màn hình *Trên máy* | An toàn với bộ nhớ nhỏ |
| **5. Hoàn thiện** | Changes API (sync tăng dần), chế độ `on_demand`/`depth`, *Mới trên Drive*, ghép đôi qua điện thoại (đăng nhập + thêm nguồn qua LAN), tự tắt Wi-Fi, Dispatcher, đổi tên/chuyển file theo Drive | Dùng hằng ngày thoải mái |
| **6. Truyện tranh** | Chế độ "Folder ảnh = truyện" + đóng gói .cbz, "Tải N chương tiếp theo", chuyển cbr/cb7 sang cbz nếu cần, xoá chương đã đọc | Đọc truyện tranh dạng folder ảnh |
| **Sau này** | Đồng bộ tiến độ đọc/highlight nhiều thiết bị, backup lên Drive (cần scope ghi) | Kế hoạch riêng |

---

## 7. Môi trường phát triển & kiểm thử

- **Kobo không phải Android nên không có ADB.** Thay vào đó dùng **SSH server có sẵn trong KOReader** (`SSH.koplugin`, chạy dropbear, có cả `scp`/`sftp-server`), kết nối qua Wi-Fi.
- **Vòng lặp phát triển đề xuất:**
  1. Sửa code trên PC.
  2. `scp -P 2222 -r gdrive.koplugin root@<ip-kobo>:/mnt/onboard/.adds/koreader/plugins/`
  3. Khởi động lại KOReader trên máy (Menu → Exit → Restart KOReader).
  4. `ssh -p 2222 root@<ip-kobo> tail -f /mnt/onboard/.adds/koreader/crash.log` để xem log trực tiếp.
- **Cách thay thế không cần SSH:** cắm USB, chép plugin vào `E:\.adds\koreader\plugins\`, eject rồi test. Chậm hơn vì mỗi lần cắm USB máy phải thoát khỏi KOReader.
- **Emulator (tuỳ chọn):** KOReader bản Linux AppImage qua WSL2 để chạy thử UI không cần máy; plugin đặt ở `plugins/gdrive.koplugin`. Cuối cùng vẫn phải test trên Kobo thật (mạng, hiệu năng).
- **Unit test** cho module thuần logic (catalog, dựng cây từ `parents`, chống vòng lặp shortcut, LRU, tạo câu `q` gộp cha) bằng `busted`, dùng response Drive mẫu.
- **Kiểm thử trên Kobo:**
  - Nguồn có ~1000 file / ~200 folder: thời gian sync lần đầu và sync tăng dần, thời gian mở folder, RAM.
  - PDF 100–300 MB: không tràn RAM; tắt Wi-Fi giữa chừng rồi resume.
  - Offline: duyệt, tìm kiếm, mở sách, xếp hàng; bật Wi-Fi lại rồi sync.
  - Gần đầy bộ nhớ: eviction đúng, không xoá sách đang đọc/đã ghim; tiến độ đọc còn nguyên sau khi xoá rồi tải lại.
  - Nguồn bị bỏ chia sẻ, shortcut vòng lặp, file trùng ở hai nguồn, token bị thu hồi.

---

## 8. Rủi ro & cách giảm thiểu

| Rủi ro | Giảm thiểu |
|---|---|
| Refresh token hết hạn sau 7 ngày | Consent screen để **In production**; báo lỗi rõ và đăng nhập lại được ngay trên Kobo (Phase 5) |
| Client id hiện có là loại Web, không redirect được về loopback | Thêm redirect URI loopback vào client, hoặc tạo client Desktop mới |
| Google siết scope `drive.readonly` với app chưa verify | Dự phòng: service account hoặc API key + folder chia sẻ theo link |
| Sync chậm với cây lớn | Gộp cha, quét cả shared drive một lần, Changes API, chế độ `on_demand` |
| Thiết bị treo khi sync | Transaction SQLite, yield UI giữa các trang, sync bị ngắt thì chạy tiếp được |
| Wi-Fi rớt giữa chừng | `.part` + Range resume, retry/backoff |
| Mất tiến độ đọc khi sách bị xoá/đổi tên | Docsettings theo hash (5.4) |
| Nickel quét trùng sách | Thư mục `.gdrive` (5.6) |
| API KOReader thay đổi giữa các phiên bản | Ghi rõ phiên bản KOReader tối thiểu đã test |

---

## 9. Quyết định đã chốt & còn mở

**Đã chốt**

- Thiết bị: Kobo, firmware 4.38.23429, KOReader v2025.04.
- Xác thực: dùng chung OAuth client Desktop với rclone, nhưng tạo token riêng scope `drive.readonly` bằng `rclone config create`; sau đó thêm ghép đôi qua điện thoại.
- Nguồn lớn nhất ~200–300 GB, nhiều loại file: lọc ngay trong truy vấn, catalog chỉ lưu sách; hỗ trợ truyện tranh (cbz/cbr/cb7, folder ảnh) ở Phase 6.
- Phát triển và test qua SSH của KOReader (không có ADB trên Kobo).
- Thư viện: nhiều folder nguồn do user thêm (My Drive, Shared with me, Shared drives); có đọc cây thư mục.
- Hạn mức dung lượng do user cấu hình.
- Không đồng bộ ngược ở giai đoạn này.

**Còn mở**

1. Số file/folder thực tế của nguồn lớn nhất: sẽ đo được sau lần quét thử đầu tiên ở Phase 2 (hoặc chạy `rclone size gdrive:<folder>` trên PC).
2. Truyện tranh chủ yếu ở dạng nào (cbz, cbr, folder ảnh)? Câu trả lời quyết định Phase 6 có cần làm sớm không.
3. Sách nào trong `Home` (6,3 GB) có thể chuyển lên Drive để giải phóng chỗ?
