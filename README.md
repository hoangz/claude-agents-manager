<div align="center">

# Claude Agents Manager

**Quản lý toàn bộ trợ lý AI của bạn — không cần biết code.**

Thay vì sửa hàng chục file rải rác trong máy, giờ bạn chỉ cần click chuột.

</div>

---

## Mục lục

- [Cài đặt lần đầu](#cài-đặt-lần-đầu)
- [Khởi động hàng ngày](#khởi-động-hàng-ngày)
- [Tổng quan giao diện](#tổng-quan-giao-diện)
- [Hướng dẫn từng tính năng](#hướng-dẫn-từng-tính-năng)
  - [Chat với trợ lý](#-chat-với-trợ-lý)
  - [Quản lý Agents](#-quản-lý-agents)
  - [Tạo Commands](#-tạo-commands)
  - [Quản lý Skills](#-quản-lý-skills)
  - [Sơ đồ kết nối](#-sơ-đồ-kết-nối)
  - [Workflow Builder](#-workflow-builder)
  - [Terminal tích hợp](#-terminal-tích-hợp)
  - [Khám phá Templates](#-khám-phá-templates)
  - [Cài đặt hệ thống](#-cài-đặt-hệ-thống)
- [Câu hỏi thường gặp](#câu-hỏi-thường-gặp)

---

## Cài đặt lần đầu

> Chỉ cần làm **một lần duy nhất**.

**Bước 1** — Cài Bun (công cụ chạy ứng dụng):
```bash
curl -fsSL https://bun.sh/install | bash
```

Đóng Terminal lại, mở lại Terminal mới.

**Bước 2** — Tải ứng dụng về máy:
```bash
git clone https://github.com/hoangz/claude-agents-manager.git
cd claude-agents-manager
```

**Bước 3** — Cài các gói cần thiết:
```bash
bun install
```

**Bước 4** — Chạy ứng dụng:
```bash
bun run dev
```

**Bước 5** — Mở trình duyệt, truy cập:
```
http://localhost:3000
```

Ứng dụng sẽ tự động đọc tất cả cấu hình Claude từ thư mục `~/.claude/` trên máy bạn.

---

## Khởi động hàng ngày

Mỗi lần muốn dùng, mở Terminal và chạy:

```bash
cd claude-agents-manager
bun run dev
```

Sau đó mở trình duyệt tại **http://localhost:3000**.

Để tắt: nhấn `Ctrl + C` trong Terminal.

---

## Tổng quan giao diện

Thanh điều hướng bên trái gồm các mục chính:

| Mục | Dùng để làm gì |
|-----|----------------|
| **Chat** | Nhắn tin với trợ lý AI |
| **Agents** | Tạo và quản lý các trợ lý |
| **Commands** | Tạo lệnh tắt cho các tác vụ lặp lại |
| **Skills** | Thêm kỹ năng cho trợ lý |
| **Graph** | Xem bản đồ kết nối tổng thể |
| **Workflows** | Xây dựng pipeline nhiều bước |
| **CLI** | Terminal tích hợp trong trình duyệt |
| **Explore** | Tải templates từ cộng đồng |
| **Settings** | Cài đặt hệ thống |

---

## Hướng dẫn từng tính năng

### 💬 Chat với trợ lý

**Vào:** menu **Chat**

Giao diện nhắn tin trực tiếp với agents, tương tự như nhắn tin trên Zalo hay Messenger.

**Cách dùng:**
1. Chọn agent từ thanh trên (hoặc để mặc định)
2. Gõ tin nhắn vào ô phía dưới
3. Nhấn **Enter** để gửi — `Shift + Enter` để xuống dòng
4. AI trả lời ngay lập tức, có thể thấy cả quá trình "suy nghĩ"

**Lưu ý:**
- Lịch sử hội thoại được lưu tự động, có thể xem lại bất kỳ lúc nào
- Mỗi cuộc hội thoại là một "session" riêng biệt
- Bấm **New Session** để bắt đầu cuộc trò chuyện mới

---

### 🤖 Quản lý Agents

**Vào:** menu **Agents**

Agents là các trợ lý AI với vai trò và cá tính riêng. Ví dụ: "Trợ lý viết nội dung", "Chuyên gia phân tích dữ liệu", "Trợ lý lập trình"...

#### Tạo agent mới

1. Click nút **New Agent** (góc trên phải)
2. Điền thông tin:
   - **Name** — Tên trợ lý (ví dụ: `Trợ lý viết email`)
   - **Description** — Mô tả ngắn gọn nhiệm vụ
   - **Model** — Chọn sức mạnh AI:
     - `Sonnet` — Cân bằng tốc độ và chất lượng *(khuyến nghị)*
     - `Opus` — Mạnh nhất, dùng cho tác vụ phức tạp
     - `Haiku` — Nhanh nhất, phù hợp tác vụ đơn giản
   - **Instructions** — Hướng dẫn cụ thể cho trợ lý (càng chi tiết càng tốt)
3. Click **Save**

#### Chỉnh sửa agent có sẵn

1. Click vào tên agent trong danh sách
2. Thay đổi bất kỳ thông tin nào
3. Click **Save**

#### Test agent ngay trong trang

1. Mở agent bất kỳ
2. Chuyển sang tab **Studio** (bên phải)
3. Gõ tin nhắn thử → xem phản hồi và từng bước xử lý của AI

#### Xóa agent

Click vào agent → click icon thùng rác → xác nhận xóa.

> **Lưu ý:** Mỗi agent được lưu thành file `.md` trong thư mục `~/.claude/agents/`. Bạn có thể xem và sửa trực tiếp file nếu cần.

---

### ⚡ Tạo Commands

**Vào:** menu **Commands**

Commands là các lệnh tắt — gõ `/tên-lệnh` trong Claude và AI sẽ tự thực hiện một tác vụ đã được định nghĩa sẵn.

**Ví dụ thực tế:**
- `/tom-tat` → AI tóm tắt nội dung đang làm
- `/review-code` → AI đánh giá đoạn code
- `/viet-email` → AI soạn email theo mẫu

#### Tạo command mới

1. Click **New Command**
2. Điền thông tin:
   - **Name** — Tên lệnh (không dấu, không cách)
   - **Description** — Lệnh này làm gì
   - **Argument hint** — Gợi ý tham số đầu vào (nếu cần), ví dụ: `[nội dung cần tóm tắt]`
   - **Allowed tools** — Các quyền AI được phép dùng khi thực thi lệnh
   - **Agent** — Liên kết với agent cụ thể (tùy chọn)
   - **Body** — Nội dung prompt đầy đủ của lệnh
3. Click **Save**

> Commands được lưu vào `~/.claude/commands/`. Có thể tổ chức theo thư mục con để dễ quản lý.

---

### 🧠 Quản lý Skills

**Vào:** menu **Skills**

Skills là các "kỹ năng" bổ sung có thể gắn vào nhiều agent khác nhau. Thay vì viết lại cùng một hướng dẫn cho nhiều agent, bạn tạo một skill và gắn vào bất kỳ agent nào cần.

#### Tạo skill mới

1. Click **New Skill**
2. Điền tên, mô tả và nội dung skill
3. Click **Save**

#### Import skill từ GitHub

1. Click **Import from GitHub**
2. Dán URL của repo GitHub (repo phải có file `SKILL.md` bên trong)
3. Xem danh sách skills tìm thấy trong repo
4. Chọn những skills muốn cài → Click **Import**
5. Skills được tải về `~/.claude/skills/`

#### Gắn skill vào agent

1. Vào trang chỉnh sửa agent
2. Ở mục **Skills**, chọn skill muốn gắn
3. Save agent

---

### 🔗 Sơ đồ kết nối

**Vào:** menu **Graph**

Bản đồ trực quan hiển thị toàn bộ mối quan hệ trong hệ thống — agent nào đang dùng skill nào, command nào liên kết agent nào.

**Cách đọc sơ đồ:**
- Mỗi ô vuông/tròn là một thực thể (agent, skill, command...)
- Đường nối thể hiện mối quan hệ phụ thuộc
- Màu sắc khác nhau phân biệt loại thực thể

**Dùng để:**
- Kiểm tra tổng thể khi có nhiều agents
- Phát hiện agent hoặc skill bị "cô lập" (không kết nối gì)
- Hiểu nhanh cấu trúc hệ thống khi mới tiếp quản

Click vào bất kỳ node nào để xem chi tiết và điều hướng đến trang chỉnh sửa.

---

### 🔄 Workflow Builder

**Vào:** menu **Workflows**

Workflow là chuỗi nhiều bước — mỗi bước do một agent khác nhau thực hiện, kết quả bước này là đầu vào bước tiếp theo.

**Ví dụ:** `Thu thập dữ liệu` → `Phân tích` → `Viết báo cáo` → `Gửi email`

#### Tạo workflow mới

1. Click **New Workflow**
2. Đặt tên và mô tả
3. Click **Add Step** để thêm từng bước
4. Mỗi bước: chọn agent thực hiện + đặt nhãn mô tả
5. Kéo thả để sắp xếp thứ tự
6. Click **Save**

#### Chạy workflow

1. Mở workflow
2. Click **Run**
3. Theo dõi kết quả từng bước theo thời gian thực

---

### 🖥️ Terminal tích hợp

**Vào:** menu **CLI**

Terminal đầy đủ ngay trong trình duyệt — không cần mở cửa sổ Terminal riêng. Có hai chế độ:

#### Chế độ Terminal

Giao diện dòng lệnh như Terminal thông thường, nhưng có thêm bảng thông tin bên phải hiển thị:

- **Tokens** — Số token đang dùng và chi phí ước tính theo thời gian thực
- **Files** — Các file đang được AI đọc/ghi
- **Tools** — Timeline các công cụ AI đã sử dụng
- **History** — Lịch sử các sessions trước

**Cách dùng:**
1. Chọn agent (tùy chọn) từ dropdown phía trên
2. Click **Start Session**
3. Gõ lệnh như Terminal thông thường
4. Xem thông tin chi tiết ở bảng bên phải

#### Chế độ Chat

Giao diện nhắn tin, tương tự mục Chat nhưng nằm trong trang CLI.

Chuyển đổi giữa hai chế độ bằng tab **Terminal** / **Chat** phía trên.

> **Lưu ý:** Lịch sử sessions được lưu tại `~/.claude/cli-history/`. Sessions tự động đóng sau 30 phút không hoạt động.

---

### 🌍 Khám phá Templates

**Vào:** menu **Explore**

Thư viện templates và agents có sẵn từ cộng đồng — cài về và dùng ngay.

**Cách cài template:**
1. Browse danh sách
2. Click vào template muốn xem chi tiết
3. Click **Install**
4. Template được tải về `~/.claude/` và xuất hiện ngay trong danh sách agents/skills

Sau khi cài, có thể vào **Agents** chỉnh sửa lại cho phù hợp nhu cầu riêng.

---

### ⚙️ Cài đặt hệ thống

**Vào:** menu **Settings**

Quản lý cấu hình toàn bộ ứng dụng:

- **Plugins** — Bật/tắt các plugin đang cài
- **MCP Servers** — Thêm hoặc xóa MCP server (kết nối Claude với công cụ bên ngoài)
- **Marketplace Sources** — Thêm nguồn template tùy chỉnh
- **Status Line** — Cài đặt thanh trạng thái Claude Code

---

## Câu hỏi thường gặp

**Dữ liệu lưu ở đâu?**
Tất cả lưu trong thư mục `~/.claude/` trên máy bạn. Không có server ngoài, không cloud — hoàn toàn trên máy cá nhân.

```
~/.claude/
├── agents/           ← Các trợ lý AI
├── commands/         ← Lệnh tắt
├── skills/           ← Kỹ năng bổ sung
├── workflows/        ← Pipeline nhiều bước
└── settings.json     ← Cài đặt chung
```

**Tắt ứng dụng có mất dữ liệu không?**
Không. Mọi thứ đã được lưu vào file ngay khi bạn nhấn Save.

**Thay đổi trong UI có ảnh hưởng đến Claude Code CLI không?**
Có — vì cả hai đều đọc cùng thư mục `~/.claude/`. Thay đổi trong UI sẽ có hiệu lực ngay với Claude Code CLI và ngược lại.

**Ứng dụng báo lỗi "port 3000 đã được dùng"?**
Chạy lệnh sau để tắt process cũ, sau đó thử lại:
```bash
lsof -ti:3000 | xargs kill -9
```

**Muốn dùng thư mục `.claude` khác?**
Tạo file `.env` trong thư mục ứng dụng:
```
CLAUDE_DIR=/đường/dẫn/tới/thư-mục-claude
```

---

## Thông tin kỹ thuật

| | |
|---|---|
| **Nền tảng** | Nuxt 3 + Vue 3 |
| **Giao diện** | Nuxt UI + Tailwind CSS |
| **Đồ thị** | VueFlow |
| **Terminal** | xterm.js + node-pty |
| **AI SDK** | @anthropic-ai/claude-agent-sdk |
| **License** | MIT |

---

<div align="center">

Fork từ [claude-code-agents-ui](https://github.com/Ngxba/claude-code-agents-ui) bởi [@Ngxba](https://github.com/Ngxba)
Dựa trên [agents-ui](https://github.com/davidrodriguezpozo/agents-ui) bởi [@davidrodriguezpozo](https://github.com/davidrodriguezpozo)

</div>
