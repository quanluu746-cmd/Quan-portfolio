# Triển khai Web 3 Tier sử dụng docker

## Chuẩn bị và kết nối AWS EC2

- chuẩn bị server: bao gồm name, AMI, - Instance type, Key pair, - Network settings và - Storage
- Kết nối SSH: `ssh -i <key của mình> ubuntu <IP công khai>`

## Cài đặt Docker trên Server
- Cập nhật hệ thống: `sudo apt update && sudo apt upgrade -y`
- Cài đặt Docker:
```
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```
- Cấp quyền chạy Docker không cần `sudo`:
```
sudo usermod -aG docker $USER
newgrp docker
```

- Kiểm tra `docker ps`

## Tạo Source Code & Cấu hình
- tạo mục dự án:

```
mkdir ~/app
cd ~/app
```
### Backend
- Chuẩn bị Backend: `mkdir backend`
- Tạo file backend/package.json: `nano backend/package.json`
```
  {
  "name": "backend",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.6.0"
  }
}
```
- Tạo file backend/index.js: `nano backend/index.js`
```
const express = require('express');
const mysql = require('mysql2');
const app = express();
const port = 5000;

const connection = mysql.createPool({
  host: process.env.DB_HOST || 'mysql-db',
  user: process.env.DB_USER || 'root',
  password: process.env.DB_PASSWORD || 'password',
  database: process.env.DB_NAME || 'testdb'
});

app.get('/', (req, res) => {
  res.send('Hello from Backend Container!');
});

app.get('/api/users', (req, res) => {
  connection.query('SELECT "User1, User2" as users', (err, results) => {
    if (err) return res.status(500).send(err);
    res.json(results);
  });
});

app.listen(port, () => {
  console.log(`Backend running on port ${port}`);
});
```
- Tạo file backend/Dockerfile: `nano backend/Dockerfile`
```
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "index.js"]
```
### Frontend
- Chuẩn bị Frontend: `mkdir -p frontend/src frontend/public`
- Tạo frontend/package.json: `nano frontend/package.json`
```
{
  "name": "frontend",
  "version": "0.1.0",
  "scripts": {
    "build": "mkdir -p build && cp -r public/* build/"
  }
}
```
- Tạo file frontend/public/index.html: `nano frontend/public/index.html`
```
<!DOCTYPE html>
<html>
<head><title>My Docker 3-Tier App</title></head>
<body>
    <h1>Hello! This is Frontend served by Nginx inside Docker.</h1>
    <p>Call /api/users to test connection to Backend.</p>
</body>
</html>
```
- Tạo file frontend/nginx.conf: `nano frontend/nginx.conf`
```
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
}
```
- Tạo file frontend/Dockerfile: `nano frontend/Dockerfile`
```
FROM node:18-alpine as builder
WORKDIR /app
COPY package.json ./
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
### Nginx proxy
- Chuẩn bị Nginx: `mkdir nginx`
- Tạo file nginx/nginx.conf: `nano nginx/nginx.conf`
```
events {}
http {
    server {
        listen 80;
        
        location /api {
            proxy_pass http://backend:5000;
        }

        location / {
            proxy_pass http://frontend:80;
        }
    }
}
```
## Gom thành file Docker Compose hoàn chỉnh
- quay lại thư mục `app` và tạo file Docker Compose: `nano docker-compose.yml`
```
version: '3.8'

services:

  mysql-db:
    image: mysql:8.0
    container_name: my_mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: testdb
      MYSQL_USER: user
      MYSQL_PASSWORD: password
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app-net

  backend:
    build: ./backend
    container_name: my_backend
    restart: always
    environment:
      DB_HOST: mysql-db
      DB_USER: user
      DB_PASSWORD: password
      DB_NAME: testdb
    depends_on:
      - mysql-db
    networks:
      - app-net

  frontend:
    build: ./frontend
    container_name: my_frontend
    restart: always
    networks:
      - app-net

  nginx-proxy:
    image: nginx:alpine
    container_name: my_proxy
    restart: always
    ports:
      - "80:80"  # Mở port 80 ra ngoài internet
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend
      - backend
    networks:
      - app-net

volumes:
  db_data:

networks:
  app-net:
    driver: bridge
```
## Chạy,kiểm tra và truy cập
- Chạy hệ thống: `docker compose up -d --build`
- Kiểm tra: `docker compose ps`
- truy cập: mở trình duyệt và nhập
- http://<IP công khai>
