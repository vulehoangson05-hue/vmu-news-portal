# Website Tin tức Trường Đại học Hàng hải Việt Nam (VIMARU News Portal)

> Đồ án môn học DevOps — Triển khai một ứng dụng web có sẵn, theo kiến trúc **microservice (Frontend / Backend tách rời)**, theo lộ trình DevOps hoàn chỉnh: từ quản lý mã nguồn, containerization, đến triển khai production có tên miền thật.

## 1. Giới thiệu

**Lưu ý quan trọng:** Source code của ứng dụng **đã được sinh sẵn/cung cấp riêng**, không thuộc phạm vi làm việc của sinh viên trong đồ án này. Tài liệu này chỉ **mô tả** source code đó để sinh viên hiểu ứng dụng mình sẽ triển khai gồm những gì. **Nhiệm vụ duy nhất của sinh viên là DEPLOY** ứng dụng theo đúng thang điểm ở mục 6 (push GitHub → Dockerfile cho từng service → Windows → Linux → online có tên miền).

Ứng dụng là một website thông tin điện tử mô phỏng cổng thông tin của **Trường Đại học Hàng hải Việt Nam (Vietnam Maritime University - VIMARU)**, gồm:

- Giới thiệu thông tin chung về nhà trường (lịch sử hình thành, sứ mệnh, tầm nhìn, cơ cấu tổ chức, các khoa/viện đào tạo, cơ sở vật chất, liên hệ...).
- Đăng tải và hiển thị **tin tức nhà trường** (thông báo tuyển sinh, hoạt động sinh viên, sự kiện, hợp tác quốc tế...).
- Quản trị viên có thể **thêm / sửa / xóa** tin tức thông qua giao diện quản trị, gọi qua API.

Trọng tâm của đồ án **không phải** là viết code hay độ phức tạp nghiệp vụ, mà là **quy trình DevOps**: đóng gói và đưa một hệ thống **nhiều service** (frontend + backend) có sẵn từ máy local lên môi trường production thực tế, có domain, có thể truy cập công khai.

## 2. Kiến trúc hệ thống (Microservice)

Ứng dụng được tách thành **3 service độc lập**, mỗi service có Dockerfile, có thể build/deploy riêng, giao tiếp với nhau qua HTTP (REST API):

```
   Người dùng
   (domain/IP thật)
        │
        ├────────────▶ port 80  ──▶ ┌────────────────────┐
        │                            │  FRONTEND SERVICE  │
        │                            │  (React/Vue/static)│──▶ gọi API_BASE_URL, AUTH_BASE_URL
        │                            │  container riêng    │
        │                            └────────────────────┘
        │
        ├────────────▶ port 4000 ─▶ ┌────────────────────┐
        │                            │  BACKEND SERVICE    │
        │                            │  (Node.js/Express   │
        │                            │   REST API)          │──▶ xác thực JWT bằng JWT_SECRET dùng chung
        │                            │  container riêng     │      (không gọi lại auth service mỗi request)
        │                            └──────────┬──────────┘
        │                                       ▼
        │                            ┌────────────────────┐
        │                            │  data/news.json     │
        │                            │  ("database" tạm,   │
        │                            │   mount volume)      │
        │                            └────────────────────┘
        │
        └────────────▶ port 5000 ─▶ ┌────────────────────┐
                                     │  AUTH SERVICE        │
                                     │  (Node.js/Express,   │
                                     │   JWT)                │
                                     │  container riêng      │
                                     └────────────────────┘
```

- **Frontend service**: chỉ lo giao diện, gọi API của backend qua `fetch/axios` (biến môi trường `API_BASE_URL`) và gọi auth service để đăng nhập quản trị (`AUTH_BASE_URL`), không đọc/ghi file JSON trực tiếp.
- **Backend service**: expose REST API (`/api/news`, `/api/pages`...), là nơi duy nhất đọc/ghi `data/news.json`. Các endpoint thêm/sửa/xóa tin tức yêu cầu JWT hợp lệ (header `Authorization: Bearer <token>`). Đã bật CORS mở (`cors()`) nên frontend gọi thẳng sang port 4000 mà không bị chặn.
- **Auth service**: microservice riêng chỉ lo xác thực quản trị viên — nhận mật khẩu (`POST /api/auth/login`), cấp JWT ký bằng `JWT_SECRET`. Backend xác thực token bằng cách verify chữ ký với cùng `JWT_SECRET` (biến môi trường dùng chung giữa hai service), không cần gọi ngược lại auth service cho mỗi request.
- Ba service **build image riêng, chạy container riêng**, kết nối qua **Docker network** (local) hoặc truy cập trực tiếp qua **3 port công khai** (80 cho frontend, 4000 cho backend, 5000 cho auth) khi deploy thật — **không dùng reverse proxy**, giữ hạ tầng tối giản.

## 3. Công nghệ sử dụng (Tech Stack)

| Thành phần | Công nghệ đề xuất | Ghi chú |
|---|---|---|
| **Frontend service** | React (Vite) hoặc HTML/CSS/JS thuần build tĩnh | Serve qua Nginx tĩnh hoặc `serve`/Node bên trong container riêng |
| **Backend service** | Node.js (Express.js) | Expose REST API, có thể thay bằng ASP.NET Core / Flask |
| **Auth service** | Node.js (Express.js) + `jsonwebtoken` | Cấp/xác thực JWT cho thao tác quản trị, tách khỏi backend |
| Database | **File JSON** (`data/news.json`) | Chỉ backend service truy cập, đọc/ghi qua `fs` module |
| Giao tiếp FE ↔ BE ↔ Auth | REST API (JSON over HTTP) | Frontend gọi qua biến môi trường `API_BASE_URL`, `AUTH_BASE_URL`; backend/auth dùng chung `JWT_SECRET` |
| Containerization | Docker, **3 Dockerfile riêng** (`frontend/Dockerfile`, `backend/Dockerfile`, `auth-service/Dockerfile`) | Mỗi service 1 image độc lập |
| Điều phối container (local/dev) | `docker-compose.yml` | Chạy đồng thời 3 service + network chung |
| Quản lý mã nguồn | Git + GitHub (monorepo hoặc nhiều repo) | Lưu trữ, versioning, cộng tác |
| Truy cập khi deploy thật | Trực tiếp qua 3 port | Frontend port 80, backend port 4000, auth port 5000, không qua reverse proxy |
| Hosting Linux | VPS (Ubuntu 22.04) | DigitalOcean / Azure / AWS EC2 / Vietnix... |
| Domain | Tên miền thật (.com/.vn/.online...) | Mua qua Namecheap, PA Vietnam, Mắt Bão... |

## 4. Cấu trúc thư mục dự kiến (monorepo, 3 service)

```
vimaru-news-portal/
├── frontend/                       # FRONTEND SERVICE (độc lập)
│   ├── src/
│   │   ├── pages/                  # Trang chủ, Giới thiệu, Tin tức, Admin
│   │   ├── components/
│   │   └── services/api.js         # Gọi API backend qua API_BASE_URL, auth qua AUTH_BASE_URL
│   ├── public/
│   ├── package.json
│   ├── .env.example                # VITE_API_BASE_URL, VITE_AUTH_BASE_URL
│   ├── Dockerfile                  # Build & serve frontend riêng
│   └── .dockerignore
│
├── backend/                        # BACKEND SERVICE (độc lập)
│   ├── src/
│   │   ├── app.js                  # Entry point Express
│   │   ├── routes/
│   │   │   ├── news.routes.js      # GET/POST/PUT/DELETE /api/news
│   │   │   └── pages.routes.js     # GET /api/pages (thông tin trường)
│   │   ├── controllers/
│   │   │   └── news.controller.js
│   │   ├── middleware/
│   │   │   └── auth.js             # Xác thực JWT (verify bằng JWT_SECRET dùng chung với auth-service)
│   │   └── services/
│   │       └── jsonDatabase.js     # Lớp đọc/ghi file JSON (giả lập DB)
│   ├── data/
│   │   └── news.json               # "Database" JSON, chỉ backend truy cập
│   ├── package.json
│   ├── Dockerfile                  # Build & chạy backend riêng
│   └── .dockerignore
│
├── auth-service/                   # AUTH SERVICE (độc lập)
│   ├── src/
│   │   ├── app.js                  # Entry point Express
│   │   ├── routes/
│   │   │   └── auth.routes.js      # POST /api/auth/login
│   │   └── controllers/
│   │       └── auth.controller.js  # Kiểm tra mật khẩu, ký JWT
│   ├── package.json
│   ├── Dockerfile                  # Build & chạy auth service riêng
│   └── .dockerignore
│
├── docker-compose.yml              # Chạy đồng thời frontend + backend + auth (local/dev/test/prod)
├── .gitignore
└── README.md
```

## 5. Chức năng chính

### 5.1. Frontend service (Public + Admin UI)
- Trang chủ giới thiệu Trường Đại học Hàng hải Việt Nam (logo, slogan, giới thiệu ngắn).
- Trang "Giới thiệu": lịch sử, sứ mệnh - tầm nhìn, các khoa/viện đào tạo, đội ngũ, liên hệ (lấy dữ liệu qua `GET /api/pages`).
- Trang "Tin tức": danh sách tin tức có phân trang, tìm kiếm (lấy dữ liệu qua `GET /api/news`).
- Trang chi tiết tin tức (`GET /api/news/:id`).
- Trang quản trị: thêm/sửa/xóa tin tức, gọi `POST/PUT/DELETE /api/news`.

### 5.2. Backend service (REST API)
| Method | Endpoint | Chức năng |
|---|---|---|
| GET | `/api/news` | Danh sách tin tức (hỗ trợ query `?page=&search=`) |
| GET | `/api/news/:id` | Chi tiết 1 tin tức |
| POST | `/api/news` | Thêm tin tức mới (cần JWT — header `Authorization: Bearer <token>`) |
| PUT | `/api/news/:id` | Sửa tin tức (cần JWT) |
| DELETE | `/api/news/:id` | Xóa tin tức (cần JWT) |
| GET | `/api/pages` | Thông tin giới thiệu nhà trường |

### 5.2b. Auth service (JWT)
| Method | Endpoint | Chức năng |
|---|---|---|
| POST | `/api/auth/login` | Đăng nhập quản trị bằng mật khẩu, trả về JWT (ký bằng `JWT_SECRET`, hết hạn sau `JWT_EXPIRES_IN`) |

Backend không gọi ngược lại auth service để kiểm tra token — cả hai service chỉ cần dùng chung một `JWT_SECRET` để backend tự verify chữ ký token.

### 5.3. "Database" bằng file JSON (chỉ backend service truy cập)
Ví dụ cấu trúc `backend/data/news.json`:
```json
{
  "news": [
    {
      "id": 1,
      "title": "Trường ĐH Hàng hải Việt Nam khai giảng năm học mới 2026-2027",
      "summary": "Sáng ngày 05/09/2026, Trường ĐH Hàng hải Việt Nam đã tổ chức lễ khai giảng...",
      "content": "Nội dung chi tiết...",
      "thumbnail": "/images/news/khai-giang-2026.jpg",
      "author": "Phòng Truyền thông",
      "publishedAt": "2026-09-05T08:00:00+07:00",
      "category": "Sự kiện"
    }
  ]
}
```
Backend đọc/ghi file này qua `services/jsonDatabase.js` (`fs.promises`). File nên được **mount ra ngoài container bằng volume** để dữ liệu không mất khi container bị xóa/rebuild.

## 6. Thang điểm / Lộ trình DevOps (tăng dần)

Đồ án được đánh giá theo các mốc triển khai tăng dần độ khó; do có **3 service**, mỗi mốc cần thực hiện cho **cả frontend, backend lẫn auth service**:

| Mốc | Yêu cầu | Mô tả |
|---|---|---|
| 1 | **Push code lên GitHub** | Repo chứa `frontend/`, `backend/` và `auth-service/`, `.gitignore` hợp lý, commit history rõ ràng |
| 2 | **Viết Dockerfile cho từng service** | `frontend/Dockerfile`, `backend/Dockerfile` và `auth-service/Dockerfile` build thành công riêng biệt, cộng thêm `docker-compose.yml` để chạy đồng thời cả 3 |
| 3 | **Deploy trên Windows** | Chạy được cả 3 container (qua `docker-compose up`) trên Windows, frontend gọi được API backend và đăng nhập qua auth service, truy cập qua `localhost` |
| 4 | **Deploy trên Linux** | Cài Docker trên máy/VM Linux (Ubuntu), `docker-compose up` chạy thành công cả 3 service, kết quả giống trên Windows |
| 5 | **Deploy online thực tế có tên miền** | Triển khai cả 3 service lên VPS/Cloud, truy cập trực tiếp qua 3 port công khai (80 cho frontend, 4000 cho backend/api, 5000 cho auth), trỏ tên miền thật về frontend |

## 7. Phạm vi công việc của sinh viên

| Đã có sẵn (không cần làm) | Sinh viên cần làm |
|---|---|
| Source code frontend + backend + auth service (giao diện, logic API, xác thực JWT, "database" JSON) | Tải/nhận source code, đọc hiểu cấu trúc 3 service để biết cách chạy & đóng gói |
| — | `git init`, tạo repo, push toàn bộ monorepo lên GitHub |
| — | Viết `Dockerfile` riêng cho `frontend/`, `backend/` và `auth-service/`, viết `docker-compose.yml` |
| — | Build & chạy cả 3 container trên **Windows** |
| — | Build & chạy cả 3 container trên **Linux** |
| — | Thuê VPS, mua domain, mở port 80/4000/5000, deploy online thực tế (không dùng reverse proxy) |

Sinh viên **không** cần sửa/viết thêm tính năng cho ứng dụng, trừ khi thiếu file cần thiết để chạy/đóng gói từng service (ví dụ thiếu `package.json`, thiếu entrypoint...) — chỉ bổ sung tối thiểu để phục vụ việc deploy, không phát triển thêm tính năng nghiệp vụ.

## 8. Cách chạy thử ứng dụng (để kiểm tra trước khi deploy)

```bash
# Auth service
cd auth-service
npm install
npm start          # chạy ở http://localhost:5000

# Backend (terminal khác)
cd backend
npm install
npm start          # chạy ở http://localhost:4000

# Frontend (terminal khác)
cd frontend
npm install
npm run dev         # chạy ở http://localhost:3000, gọi API_BASE_URL/AUTH_BASE_URL trỏ về 2 service trên
```

Hoặc chạy cả 3 cùng lúc bằng Docker Compose (khuyến nghị vì gần giống môi trường deploy thật):
```bash
docker compose up --build
```

Bước này chỉ nhằm xác nhận cả 3 service chạy được và giao tiếp đúng trước khi tiến hành các mốc deploy ở mục 6.
