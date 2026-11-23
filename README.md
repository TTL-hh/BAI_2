# BÀI 2: API với Node-RED và gọi API từ trang web

| Thông tin | Chi tiết |
|---|---|
| **Họ tên học sinh** | Trần Tùng Lâm |
| **MSSV** | K235480106039 |
| **Môn học** | Lập trình Web |
| **Môi trường thực hiện** | Windows + WSL Ubuntu |
| **Công nghệ sử dụng** | Docker, Docker Compose, Nginx, MariaDB, phpMyAdmin, Node-RED, Cloudflared |

---

## Cấu trúc thư mục

```text
bai2/
├── README.md
├── docker-compose.yml
├── nginx/conf.d/default.conf
├── nodered-data/flows.json
└── site/index.html
```

---

## Yêu cầu

1. Dùng Node-RED: node `http in` + `http response` để tạo API đơn giản.
2. Cấu hình Nginx để web dùng JavaScript gọi được API trên Node-RED; tự nghĩ thuật toán cho API.
3. Viết code JavaScript trong trang HTML để gọi API trên.

Định dạng JSON trả về theo đề:

```json
{"ok":1,"msg":"thành công","dssv":[{"name":"Cốp","money":123},{"name":"David","money":456}]}
```

## Khởi động

```bash
cd bai2
sudo chown -R 1000:1000 nodered-data
docker compose up -d
```

> Nếu đang chạy Bài 1, chạy `docker compose down` trong thư mục `bai1` trước để tránh trùng cổng 80 và 1880.

## 1. Tạo API bằng Node-RED

Mở http://localhost:1880, flow **API tacke** gồm 3 node nối tiếp:

```text
[http in: GET /api/tacke] → [function] → [http response]
```

- **http in:** Method `GET`, URL `/api/tacke`.
- **function:** tạo dữ liệu và xử lý theo thuật toán (bên dưới).
- **http response:** trả `msg.payload` về dưới dạng JSON.

**Thuật toán tự nghĩ:** từ danh sách sinh viên, API

1. Sắp xếp theo `money` giảm dần.
2. Gán thứ hạng `rank` cho từng sinh viên.
3. Tính tổng tiền của cả lớp, trả về trường `tong`.

```javascript
const dssv = [
    { name: "Cốp", money: 123 },
    { name: "David", money: 456 },
    { name: "Lan", money: 300 }
];

dssv.sort((a, b) => b.money - a.money);
dssv.forEach((sv, i) => { sv.rank = i + 1; });

const tong = dssv.reduce((s, sv) => s + sv.money, 0);

msg.payload = { ok: 1, msg: "thành công", tong: tong, dssv: dssv };
msg.statusCode = 200;
return msg;
```

Kiểm tra bằng lệnh:

```bash
curl http://localhost/api/tacke
```

Kết quả:

```json
{
  "ok": 1,
  "msg": "thành công",
  "tong": 879,
  "dssv": [
    {"name": "David", "money": 456, "rank": 1},
    {"name": "Lan",   "money": 300, "rank": 2},
    {"name": "Cốp",   "money": 123, "rank": 3}
  ]
}
```

<!-- ![Flow Node-RED](images/bai2-flow.png) -->
<!-- ![Kết quả curl](images/bai2-curl.png) -->

## 2. Cấu hình Nginx để web gọi được API

Trang web và Node-RED chạy ở hai cổng khác nhau nên nếu gọi thẳng `:1880` sẽ bị chặn bởi CORS. Giải pháp là dùng Nginx làm **reverse proxy**: web chỉ gọi đường dẫn `/api/...` cùng domain, Nginx tự chuyển tiếp sang Node-RED.

```nginx
location /api/ {
    proxy_pass http://nodered:1880/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Trong Docker Compose, Nginx gọi Node-RED qua tên dịch vụ `nodered` (mạng nội bộ của Compose).

## 3. Code JavaScript gọi API trong trang HTML

File `site/index.html`:

```html
<script>
  fetch('/api/tacke')
    .then(res => res.json())
    .then(data => {
      if (data.ok !== 1) throw new Error(data.msg);
      const ul = document.getElementById('list');
      data.dssv.forEach(sv => {
        const li = document.createElement('li');
        li.textContent = `#${sv.rank} ${sv.name}: ${sv.money}`;
        ul.appendChild(li);
      });
      document.getElementById('total').textContent = 'Tổng tiền: ' + data.tong;
    })
    .catch(err => {
      document.getElementById('status').textContent = 'Lỗi: ' + err.message;
    });
</script>
```

Mở http://localhost để xem danh sách sinh viên được tải từ API và hiển thị lên trang.

<!-- ![Trang web gọi API](images/bai2-web.png) -->

## Kết quả Bài 2

- [x] Tạo được API bằng `http in` + `http response` trên Node-RED
- [x] Nginx reverse proxy giúp web gọi API không bị lỗi CORS
- [x] Có thuật toán xử lý dữ liệu cho API (sắp xếp, xếp hạng, tính tổng)
- [x] Trang HTML dùng JavaScript `fetch` gọi và hiển thị dữ liệu từ API

---

## Lệnh hữu ích

```bash
docker compose ps                 # xem trạng thái các dịch vụ
docker compose logs -f nginx      # xem log nginx
docker compose restart nginx      # nạp lại cấu hình nginx
docker compose down               # dừng và xóa container
```
