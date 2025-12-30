# triển khai web 3 tier

## Chuẩn bị và kết nối AWS EC2

- chuẩn bị server: bao gồm name, AMI, - Instance type, Key pair, - Network settings và - Storage
- Kết nối SSH: `ssh -i <key của mình> ubuntu <IP công khai>`
- cập nhập hệ thống: `sudo apt update & sudo apt upgrade`

*Lưu ý: Do EC2 gói Free Tier chỉ có 1GB RAM. Việc cài đặt và build React cần nhiều RAM hơn thế. Nên bài này dùng giải pháp là tạo Ram ảo*

```
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```
## database

- Cài đặt MySQL server: `sudo apt install mysql-server -y`
- tạo database: ` sudo mysql`
  
  tạo database tên là ‘luuducquan’
  
  tạo user tên là ‘quanluu’, mật khẩu là ’akcyend9’
  
  cấp quyền cho user trên database
  
  áp dụng quyền
  
  thoát

```
CREATE DATABASE luuducquan;

CREATE USER 'quanluu'@'localhost' IDENTIFIED BY 'akcyend9';

GRANT ALL PRIVILEGES ON luuducquan.* TO 'quanluu'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## application
- Cài đặt Node.js: Ubuntu mặc định cài Node phiên bản rất cũ. Ta dùng curl để lấy bản mới hơn (ví dụ v20).
```
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```
- Tải mã nguồn và cài đặt thư viện:
```
mkdir ~/apps
cd ~/apps
git clone https://github.com/bezkoder/nodejs-express-mysql.git backend
cd backend
npm install
```
- cấu hình kết nối database: `nano app/config/db.config.js`

viết nội dung: user, password và database đã tạo ở trên
 ```module.exports = {
  HOST: "localhost",
  USER: "quanluu",     
  PASSWORD: "akcyend9", 
  DB: "luuducquan",         
  dialect: "mysql",
  pool: {
    max: 5,
    min: 0,
    acquire: 30000,
    idle: 10000
  }
};
```

  chạy thử xem lỗi không: `node server.js`
- cài PM2 để cạy nền:
```
sudo npm install -g pm2
pm2 start server.js --name backend-api
```

## web server
- Build frontend
```cd ~/apps
git clone https://github.com/bezkoder/react-crud-web-api.git frontend
cd frontend
npm install
```
*Cấu hình đường dẫn API: http://localhost:8080/api. Khi lên server thật, trình duyệt người dùng không thể hiểu localhost:8080 là server của bạn. Ta sẽ sửa thành đường dẫn tương đối /api.*
Mở file cấu hình http: `nano src/http-common.js`
sửa thành:
```
import axios from "axios";

export default axios.create({
  baseURL: "/api",
  headers: {
    "Content-type": "application/json"
  }
});
```
- Build ra file tĩnh: `npm run build`
- cài đặt Nginx: `sudo apt install nginx -y`
- Viết file cấu hình: `sudo nano /etc/nginx/sites-available/default`
Khai báo server
Phục vụ Frontend
Reverse Proxy (Backend)
```
server {
    listen 80;
    server_name <IP công khai>;

    root <Trỏ vào thư mục build React vừa tạo>; 
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
## phân quyền và khởi động lại
- phân quyền
```
sudo chown -R www-data:www-data <Trỏ vào thư mục build React vừa tạo>
sudo chmod -R 755 <Trỏ vào thư mục build React vừa tạo>
```
- khởi động lại
```
sudo nginx -t
sudo systemctl restart nginx
```
  
  
