# Hướng dẫn: Dùng REA + Claude Code để tìm hiểu một app trên Windows

**Mục tiêu:** hiểu app hoạt động thế nào và nó nhận gì vào (đầu vào), xuất gì ra (đầu ra).
**Phạm vi:** nghiên cứu cá nhân, chủ yếu phân tích **tĩnh**, tức là đọc file của app mà không chạy nó.

> Nguồn: hướng dẫn tổng hợp từ tài liệu trong repo REA (`README.md`,
> `docs/getting-started/installation.md`, `docs/desktop-applications.md`,
> `docs/providers/windows-ghidra-p0.md`, `docs/guides/...`). Người viết chưa chạy
> thử các bước này trên Windows. Nếu một lệnh không khớp với phiên bản REA bạn cài,
> hãy chạy `rea --help` hoặc hỏi Claude Code.

---

## Bước 0: Chuẩn bị và lưu ý

- [ ] Windows 10 trở lên, bản **64-bit (x64)**.
- [ ] Đã cài **Claude Code**.
- [ ] Biết thư mục cài đặt của app (xem Bước 2).
- [ ] Đọc nhanh `DISCLAIMER.md` của REA. Mục đích nghiên cứu là phù hợp, nhưng:
  - Giấy phép (EULA) của app hoặc luật nơi bạn ở có thể **hạn chế việc dịch ngược**.
  - REA **không hỗ trợ** vượt DRM, license check, anti-cheat, hay sao chép code độc quyền.
- [ ] **Không** đưa thông tin nhạy cảm (mật khẩu, token…) vào prompt.

---

## Bước 1: Cài Node.js và REA

### 1.1. Cài Node.js
REA yêu cầu một trong các phiên bản: **22.x (≥ 22.23.4)**, **24.x (≥ 24.22.1)** hoặc **26.12.1+**.
Tải bản **LTS** tại https://nodejs.org rồi cài như phần mềm thông thường.

Mở **PowerShell** và kiểm tra:
```powershell
node --version
npm --version
```

### 1.2. Cài REA và đăng ký với Claude Code
```powershell
npx rea-agents setup
```
- Chọn **Claude Code** khi được hỏi.
- REA hiển thị **danh sách thay đổi** sẽ thực hiện. Đọc rồi duyệt (approve).
- REA tự sao lưu cấu hình cũ trước khi ghi.
- Trên Windows, setup **không** cài Hopper, Ghidra hay Java. Đây là bình thường.

### 1.3. (Nên làm) Cài lệnh `rea` để dùng trực tiếp trong terminal
```powershell
npm install --global rea-agents
rea --help
```

### 1.4. Khởi động lại Claude Code và kiểm tra
```powershell
npx rea-agents doctor --client claude_code --skill --json
```
Mở Claude Code và hỏi:
> Bạn có thấy MCP server `rea` không? Gọi tool `binary_session` với `{}` và cho tôi biết trạng thái.

Nếu Claude Code trả lời được, REA đã sẵn sàng.

---

## Bước 2: Tìm thư mục cài đặt của app

Chọn một trong các cách:
- Chuột phải vào **shortcut** của app, chọn **Open file location**. Nếu đó vẫn là shortcut, chuột phải tiếp vào file `.lnk`, chọn **Properties**, xem ô **Target**.
- Mở **Task Manager** khi app đang chạy, chuột phải vào tiến trình, chọn **Open file location**.
- Các vị trí thường gặp:
  - `C:\Program Files\<TênApp>`
  - `C:\Program Files (x86)\<TênApp>`
  - `C:\Users\<bạn>\AppData\Local\<TênApp>` (gõ `%LOCALAPPDATA%` vào thanh địa chỉ Explorer)

Ghi lại đường dẫn. Trong prompt nên dùng dấu `/`, ví dụ `C:/Program Files/TenApp`.

---

## Bước 3: Xác định app viết bằng gì (chưa cần Ghidra)

Trong Claude Code, gửi prompt:

> Dùng REA tool `inspect_desktop_application` cho thư mục `C:/Program Files/TenApp`.
> Cho tôi biết: (1) app viết bằng công nghệ gì, kèm bằng chứng; (2) file thực thi chính
> và các thành phần quan trọng; (3) các bước phân tích tiếp theo REA gợi ý (next probes).
> Phân biệt rõ điều quan sát được và điều suy đoán.

Tool này **không chạy app**, không giải nén và không sửa file nào. Nó chỉ đọc danh sách file và header.

Nếu muốn chạy bằng terminal:
```powershell
rea inspect-desktop-application "C:/Program Files/TenApp" --detail summary --evidence_output "C:/rea-reports/app.evidence.json" --json
```
Thư mục `C:/rea-reports` phải tồn tại trước. REA **không ghi đè** file đã có, nên mỗi lần chạy hãy dùng tên file mới.

**Kết quả sẽ cho biết app thuộc loại nào. Sang Bước 4 và làm theo đúng mục của loại đó.**

---

## Bước 4: Phân tích theo loại app

### 4A. Electron / JavaScript
Dấu hiệu: có thư mục `resources`, file `app.asar`, các file `chrome_*.pak`, `ffmpeg.dll`.

Prompt:
> Dùng REA `analyze_javascript_application` cho `C:/Program Files/TenApp/resources/app.asar`.
> Liệt kê: các module chính, kênh IPC giữa giao diện và tiến trình chính, URL/endpoint
> mạng, file mà app đọc/ghi, native add-on. Kèm evidence.

Bằng terminal:
```powershell
rea analyze-javascript-application "C:/Program Files/TenApp/resources/app.asar" --json > C:/rea-reports/js.json
```
Không cần cài thêm gì.

### 4B. .NET (C#, WPF, WinForms)
Dấu hiệu: nhiều file `.dll` của .NET, file `*.runtimeconfig.json`, `*.deps.json`.

Prompt:
> Dùng REA `inspect_managed_artifact` cho `C:/Program Files/TenApp/TenApp.exe` (hoặc DLL chính).
> Sau đó dùng `inspect_managed_members` để tìm các class/method xử lý đầu vào (mở file,
> nhận dữ liệu mạng, xử lý UI) và đầu ra (ghi file, gửi request). Dùng
> `inspect_managed_native_boundaries` để xem app gọi DLL native nào.

Không cần cài thêm gì, vì REA đọc metadata và mã CIL mà không chạy code.

### 4C. Native (C/C++, Rust, Go…)
Dấu hiệu: không thuộc các loại trên; file `.exe`/`.dll` thuần.

**Bắt đầu với các tool không cần Ghidra:**
```powershell
rea inspect-pe-imports "C:/Program Files/TenApp/TenApp.exe" --json
rea inspect-binary-hardening "C:/Program Files/TenApp/TenApp.exe" --json
```
Nếu là Go hoặc Rust:
```powershell
rea inspect-go-binary "C:/Program Files/TenApp/TenApp.exe" --json
rea inspect-rust-binary "C:/Program Files/TenApp/TenApp.exe" --json
```

**Khi cần decompile thì mới cài Ghidra** (hỗ trợ Windows còn ở mức *thử nghiệm*, chỉ đọc):
1. Cài **JDK 64-bit bản đầy đủ, phiên bản 25 trở lên**, ví dụ từ https://adoptium.net.
2. Tải **Ghidra 12.2 trở lên** (12.1 **không** được chấp nhận) từ https://ghidra-sre.org rồi giải nén, ví dụ vào `C:\tools\ghidra_12.2_PUBLIC`.
3. Cấu hình rồi kiểm tra trong PowerShell:
   ```powershell
   $env:GHIDRA_INSTALL_DIR = "C:\tools\ghidra_12.2_PUBLIC"
   $env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-25"   # sửa theo đường dẫn thật
   rea doctor --json
   rea providers --json
   ```
   Để các biến này dùng được lâu dài, hãy đặt chúng trong *System Properties → Environment Variables*, hoặc chạy lại `npx rea-agents setup` để REA ghi chúng vào cấu hình của Claude Code.
4. Khởi động lại Claude Code, rồi gửi prompt:
   > Dùng REA với provider ghidra mở `C:/Program Files/TenApp/TenApp.exe`. Tìm hàm entry,
   > các hàm gọi API mạng/file/registry, rồi decompile những hàm quan trọng nhất và giải
   > thích luồng xử lý. Kèm evidence.

### 4D. Loại khác
| Dấu hiệu | Gợi ý prompt |
|---|---|
| PyInstaller (exe chứa Python) | "Dùng REA phân tích PyInstaller executable `...exe`" |
| Unity (`GameAssembly.dll`, `global-metadata.dat`) | "Dùng REA đọc IL2CPP metadata `.../global-metadata.dat`" |
| Godot (`.pck`) | "Dùng REA kiểm tra Godot PCK `...pck`" |
| Deno compile | "Dùng REA phân tích Deno compile executable `...exe`" |

---

## Bước 5: Tìm đầu vào và đầu ra

Sau Bước 4, gửi prompt tổng hợp:

> Dựa trên evidence REA đã thu thập trong phiên này, hãy lập bảng:
> **ĐẦU VÀO**: app nhận dữ liệu từ đâu (người dùng, file, mạng, registry, tham số dòng lệnh, IPC).
> **XỬ LÝ**: các module/hàm chính và luồng đi của dữ liệu.
> **ĐẦU RA**: app ghi/gửi gì (file, mạng, registry, giao diện).
> Với mỗi mục, ghi evidence_id và phân loại: *quan sát*, *suy luận*, hay *chưa biết*.

Cách đọc các bằng chứng thường gặp:
| Bằng chứng | Gợi ý về |
|---|---|
| Import `ws2_32.dll`, `winhttp.dll`, `wininet.dll` | Giao tiếp mạng |
| Import `CreateFileW`, `ReadFile`, `WriteFile` | Đọc/ghi file |
| Import `RegOpenKeyExW`, `RegSetValueExW` | Đọc/ghi registry |
| Strings chứa `https://...` | Endpoint server |
| Strings như `.json`, `.ini`, `config` | File cấu hình |
| Kênh IPC (Electron) | Cách giao diện nói chuyện với lõi app |

**Giới hạn quan trọng:** đây là phân tích **tĩnh**. Nó cho biết app *có khả năng* làm gì, không phải những gì app *thực sự* làm trong một lần chạy cụ thể. REA **chưa hỗ trợ ghi lại hành vi lúc chạy (process capture) trên Windows native**.

---

## Bước 6 (tùy chọn): Quan sát app lúc đang chạy

Phần này nằm **ngoài REA**. Bạn có thể dùng **Process Monitor** (Microsoft Sysinternals,
https://learn.microsoft.com/sysinternals/downloads/procmon) để xem app thực sự mở file nào
và đọc/ghi registry nào khi bạn thao tác:
1. Chạy Procmon và đặt filter `Process Name` `is` `TenApp.exe`.
2. Thao tác tính năng muốn tìm hiểu trên app.
3. Xuất log và đối chiếu với kết quả tĩnh ở Bước 5.

> REA có các tool Frida (`start_frida_session`…) để gắn vào process đang chạy, nhưng tài liệu
> REA **chưa xác nhận hỗ trợ trên Windows** (bước kiểm chứng của dự án chỉ chạy trên Linux),
> và script Frida chạy *bên trong* app với toàn quyền của app. Chỉ thử trên máy ảo hoặc máy thử nghiệm.

---

## Bước 7: Lưu và xem lại kết quả

- Trong Claude Code:
  > Dùng `get_evidence_bundle` với detail summary để liệt kê evidence đã thu trong phiên này.
- Để lưu toàn bộ bằng chứng ra file:
  > Dùng `export_evidence_bundle` để xuất toàn bộ evidence ra `C:/rea-reports/bundle.json`.
- Phiên MCP kết thúc khi Claude Code tắt, nên **hãy xuất bundle trước khi đóng**.

---

## Xử lý sự cố nhanh

| Vấn đề | Cách xử lý |
|---|---|
| Claude Code không thấy REA | Khởi động lại Claude Code; chạy `npx rea-agents doctor --client claude_code --skill --json` |
| Báo lỗi phiên bản Node | Cài đúng phiên bản ở Bước 1.1 |
| Ghidra không được nhận | Kiểm tra Ghidra ≥ 12.2, JDK ≥ 25 bản 64-bit đầy đủ; chạy `rea doctor --json` |
| Ghidra chạy lâu với file lớn | Tăng `REA_GHIDRA_STARTUP_TIMEOUT_MS` |
| Lệnh báo file đầu ra đã tồn tại | REA không ghi đè, nên đổi tên file đầu ra |
| Bị Windows Defender chặn | Kiểm tra *Protection history* của Defender trước khi kết luận lỗi do REA |

---

## Checklist tóm tắt
- [ ] B1: Cài Node.js → `npx rea-agents setup` → khởi động lại Claude Code
- [ ] B2: Tìm thư mục cài đặt app
- [ ] B3: `inspect_desktop_application` để biết loại app
- [ ] B4: Phân tích theo loại (chỉ cài Ghidra/JDK nếu là native và cần decompile)
- [ ] B5: Lập bảng đầu vào → xử lý → đầu ra kèm evidence
- [ ] B6: (tùy chọn) Procmon để đối chiếu hành vi thực tế
- [ ] B7: Xuất evidence bundle trước khi đóng phiên
