# TẠO WEB SERVER

## chuẩn bị và kết nối vào môi trường AWS EC2

- chuẩn bị server: bao gồm name, AMI, -	Instance type, Key pair, -	Network settings và -	Storage
- Kết nối SSH: `ssh -i <key của mình> ubuntu <IP công khai>`
- cập nhập hệ thống: `sudo apt update`
- cài đặt Nginx: `sudo apt install nginx git -y`

## tải và cấu hình website pỏtfolio

- Di chuyển đến thư mục web: `cd ~/projects`
- Tải repository mẫu: `sudo git clone [https://github.com/codewithsadee/vcard-personal-portfolio]`
- Thay thế tên thư mục: `sudo mv vcard-personal-portfolio /var/www/`
- Đặt quyền sở hữu cho Nginx
   
    `sudo chown -R www-data:www-data /var/www/vcard-personal-portfolio`

    `sudo find /var/www/vcard-personal-portfolio -type d -exec chmod 755 {} \;`

    `sudo find /var/www/vcard-personal-portfolio -type f -exec chmod 644 {} \;`

## tạo mội file cấu hình mới

- trong thư mục /etc/nginx/sites-available/. đặt tên file là "portfolio.conf": `sudo nano /etc/nginx/sites-available/portfolio.conf`

    Lắng nghe trên cổng 80 (cổng HTTP tiêu chuẩn)
  
    Đặt tên cho server
  
    Định nghĩa thư mục gốc của trang web
  
    Thứ tự tìm kiếm file index khi truy cập thư mục
  
    Thiết lập log files
  
    Cấu hình cơ bản để phục vụ các file

 ```server {
    
    listen 80;
    listen [::]:80;
 
    server_name <IP công khai>;
    
    root /var/www/vcard-personal-portfolio;
    
    index index.html index.htm;
    
    access_log /var/log/nginx/portfolio.access.log;
    error_log /var/log/nginx/portfolio.error.log;
 
    location / {
        try_files $uri $uri/ =404;
    }
}
```

## Kiểm tra, khởi động lại Nginx và truy cập

- Kiểm tra: `sudo nginx -t`
- Khởi động lại đẻ áp dụng thay đổi: `sudo systemctl restart nginx`
- mở trình duyệt và nhập `http://<IP công khai>`
