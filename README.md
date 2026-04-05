<div align="center">

# Claude Agents Manager

**Quản lý toàn bộ trợ lý AI của bạn — không cần biết code.**

Giao diện trực quan để tạo, chỉnh sửa và vận hành hệ thống Claude agents, workflows, skills, commands và MCP servers — tất cả trong một nơi.

[![MIT License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Nuxt 3](https://img.shields.io/badge/Nuxt-3-00DC82.svg)](https://nuxt.com)
[![Bun](https://img.shields.io/badge/runtime-Bun-fbf0df.svg)](https://bun.sh)

</div>

---

## Mục lục

- [Cài đặt lần đầu](#cài-đặt-lần-đầu)
- [Khởi động hàng ngày](#khởi-động-hàng-ngày)
- [Tổng quan giao diện](#tổng-quan-giao-diện)
- [Hướng dẫn từng tính năng](#hướng-dẫn-từng-tính-năng)
- [Câu hỏi thường gặp](#câu-hỏi-thường-gặp)
- [Changelog](#changelog)

---

## Cài đặt lần đầu

> Chỉ cần làm **một lần duy nhất**.

**Bước 1** — Cài Bun (công cụ chạy ứng dụng):
```bash
curl -fsSL https://bun.sh/install | bash
```
Đóng Terminal lại, mở lại Terminal mới.

**Bước 2** — Tải ứng dụng:
```bash
git clone https://github.com/hoangz/claude-agents-manager.git
cd claude-agents-manager
```

**Bước 3** — Cài dependencies:
```bash
bun install
```

**Bước 4** — Chạy ứng dụng:
```bash
bun run dev
```

**Bước 5** — Mở trình duyệt:
```
http://localhost:3000
```

Ứng dụng tự động đọc cấu hình Claude từ `~/.claude/` trên máy bạn.

---

## Khởi động hàng ngày

```bash
cd claude-agents-manager
bun run dev
```

Truy cập **http://localhost:3000** — nhấn `Ctrl + C` để tắt.

---

## Tổng quan giao diện

Thanh điều hướng bên trái:

| Mục | Dùng để làm gì |
|-----|----------------|
| **Dashboard** | Tổng quan hệ thống |
| **Workflows** | Xây dựng pipeline nhiều bước |
| **Agents** | Tạo và quản lý trợ lý AI |
| **Skills** | Quản lý kỹ năng cho agents |
| **Plugins** | Quản lý plugin |
| **Commands** | Tạo lệnh tắt `/command` |
| **MCP Servers** | Kết nối công cụ bên ngoài |
| **Graph** | Bản đồ kết nối tổng thể |
| **Health** | Đánh giá sức khoẻ hệ thống |
| **History** | Lịch sử phiên làm việc |
| **CLI** | Terminal tích hợp |
| **Explore** | Tải templates từ cộng đồng |
| **Settings** | Cài đặt & sao lưu dữ liệu |

> Mỗi trang có nút **Hướng dẫn** ở góc trên phải — click để xem hướng dẫn sử dụng ngay trong trang.

---

## Hướng dẫn từng tính năng

### 🔄 Workflow Builder

Xây dựng pipeline nhiều bước — mỗi bước do một agent thực hiện, kết quả bước này là đầu vào bước tiếp theo.

**Ví dụ:** `Thu thập dữ liệu` → `Phân tích` → `Viết báo cáo` → `Gửi email`

1. Click **New Workflow** → đặt tên và mô tả
2. Click **Add Step** → chọn agent + đặt nhãn cho từng bước
3. Kéo thả để sắp xếp thứ tự
4. Click **Save** → Click **Run** để chạy và theo dõi kết quả từng bước

---

### 🤖 Quản lý Agents

Agents là trợ lý AI với vai trò riêng. Ví dụ: "Trợ lý viết nội dung", "Chuyên gia phân tích dữ liệu"...

#### Tạo agent mới
1. Click **New Agent** (góc trên phải)
2. Điền **Name**, **Description**, chọn **Model**:
   - `Sonnet` — Cân bằng tốc độ / chất lượng *(khuyến nghị)*
   - `Opus` — Mạnh nhất, cho tác vụ phức tạp
   - `Haiku` — Nhanh nhất, tác vụ đơn giản
3. Điền **Instructions** (càng chi tiết càng tốt) → **Save**

#### Phân loại agents
Agents tự động được phân vào các nhóm:
- **Coding** — lập trình, debug, review code
- **Writing** — viết nội dung, blog, email
- **Analysis** — phân tích dữ liệu, báo cáo
- **Design** — UI/UX, thiết kế
- **DevOps** — deploy, hạ tầng, CI/CD
- **Manager** — quản lý dự án, điều phối
- **General** — tổng hợp

Dùng filter pills để lọc theo nhóm.

#### Xóa nhiều agents cùng lúc
Hover vào agent → tick ô checkbox → chọn thêm → thanh **Bulk Actions** hiện ở dưới → **Delete Selected**.

---

### 🧠 Quản lý Skills

Skills là kỹ năng bổ sung, gắn vào nhiều agent khác nhau.

#### Phân loại skills
- **Development** · **Writing** · **Analysis** · **Design** · **DevOps** · **General**

Hỗ trợ **List / Grid** view, sắp xếp theo tên hoặc ngày tạo.

#### Import từ GitHub
1. Click **Import from GitHub**
2. Dán URL repo (phải có file `SKILL.md`)
3. Chọn skills muốn cài → **Import**

---

### ⚡ Tạo Commands

Commands là lệnh tắt — gõ `/tên-lệnh` trong Claude để thực thi tác vụ định sẵn.

1. Click **New Command**
2. Điền **Name** (không dấu, không cách), **Description**, **Body** (nội dung prompt)
3. Tuỳ chọn: **Argument hint**, **Allowed tools**, liên kết **Agent**
4. **Save** — lệnh sẵn sàng dùng trong Claude Code CLI

---

### 🔗 Sơ đồ kết nối (Graph)

Bản đồ trực quan toàn bộ hệ thống:

- **Tìm kiếm** — gõ tên để highlight node
- **Filter pills** — lọc theo loại: Agents / Commands / Skills / Plugins / MCP
- **Active Only** — ẩn node không có kết nối, chỉ hiện những gì đang hoạt động
- Click node để xem chi tiết và điều hướng đến trang chỉnh sửa

---

### 📋 Lịch sử hoạt động (History)

Xem lại toàn bộ lịch sử sessions Claude CLI theo dự án:
- Duyệt sessions theo dự án, thời gian tương đối
- Tìm kiếm theo tên hoặc nội dung
- Xem model AI đã dùng trong mỗi session

---

### 🏥 Sức khoẻ hệ thống (Health)

Tự động phân tích cấu hình và đưa ra điểm số 0–100:
- **80–100** 🟢 Khoẻ mạnh
- **50–79** 🟡 Cần chú ý
- **0–49** 🔴 Cần cải thiện

Filter: **All / Warnings / Info** — Click **Fix →** để đến thẳng trang cần sửa.

---

### 🖥️ Terminal tích hợp (CLI)

Terminal đầy đủ trong trình duyệt, có 2 chế độ:
- **Terminal** — dòng lệnh với bảng info real-time (tokens, files, tools, history)
- **Chat** — giao diện nhắn tin

---

### ⚙️ Cài đặt & Sao lưu (Settings)

#### Export (Sao lưu)
**Settings** → **Data Management** → **Export Config** → tải file JSON chứa toàn bộ agents, commands, skills, workflows.

#### Import (Khôi phục)
**Import Config** → kéo thả file JSON → xem trước → chọn ghi đè hay không → **Import**.

---

## Câu hỏi thường gặp

**Dữ liệu lưu ở đâu?**
```
~/.claude/
├── agents/       ← Trợ lý AI
├── commands/     ← Lệnh tắt
├── skills/       ← Kỹ năng bổ sung
├── workflows/    ← Pipeline nhiều bước
└── settings.json ← Cài đặt chung
```
Tất cả trên máy cá nhân, không cloud, không server ngoài.

**Thay đổi trong UI có ảnh hưởng đến Claude Code CLI không?**
Có — cả hai đọc cùng `~/.claude/`. Thay đổi có hiệu lực ngay.

**Lỗi "port 3000 đã được dùng"?**
```bash
lsof -ti:3000 | xargs kill -9
```

**Muốn dùng thư mục `.claude` khác?**
Tạo file `.env`:
```
CLAUDE_DIR=/đường/dẫn/tới/thư-mục-claude
```

---

## Changelog

### v2.0.0 — UI Redesign & New Features

#### 🎨 Giao diện
- **Theme**: Figma / Notion Dark — nền ấm `#1e1e1e`, accent teal `#00c9a7`
- **Font**: Chuyển sang **Inter** (font Facebook/Meta) — heading đậm 700–800
- **Border radius**: Bo tròn hơn (6/8/12px)

#### 🆕 Tính năng mới
- **History page** — xem lịch sử sessions Claude CLI theo dự án
- **Health page** — health score 0–100, gợi ý cải thiện, filter Warnings/Info
- **Export / Import config** — backup toàn bộ cấu hình ra file JSON, import lại
- **Bulk delete agents** — chọn nhiều agent và xóa cùng lúc
- **Agents categorization** — tự động phân nhóm Coding/Writing/Analysis/Design/DevOps/Manager/General + filter pills
- **Skills categorization** — tự động phân nhóm + List/Grid view + sort
- **Help Guide** — nút Hướng dẫn trên mỗi trang, hướng dẫn từng bước cho người dùng mới

#### 📊 Graph
- Search node theo tên
- Filter pills theo loại (Agents / Commands / Skills / Plugins / MCP)
- Toggle **Active Only** — ẩn node không kết nối
- Cột cố định: Commands → Agents → Skills → Plugins → MCP
- Edge colors phân biệt 3 loại quan hệ

#### 🧭 Navigation
- Thứ tự sidebar mới: Dashboard → Workflows → Agents → Skills → Plugins → Commands → MCP Servers
- Thêm History và Health vào secondary nav

---

## Thông tin kỹ thuật

| | |
|---|---|
| **Nền tảng** | Nuxt 3 + Vue 3 |
| **Giao diện** | Nuxt UI + Tailwind CSS |
| **Font** | Inter + Fira Code |
| **Đồ thị** | VueFlow |
| **Terminal** | xterm.js + node-pty |
| **AI SDK** | @anthropic-ai/claude-agent-sdk |
| **Runtime** | Bun |
| **License** | MIT |

---

<div align="center">

Fork từ [claude-code-agents-ui](https://github.com/Ngxba/claude-code-agents-ui) bởi [@Ngxba](https://github.com/Ngxba)
Dựa trên [agents-ui](https://github.com/davidrodriguezpozo/agents-ui) bởi [@davidrodriguezpozo](https://github.com/davidrodriguezpozo)

</div>
