Cài đặt Teleport

Teleport có 2 bản Enterprise và Community Edition (Miễn phí)
Bước 1: Cài đặt
```
curl https://cdn.teleport.dev/install.sh | bash -s "18.7.3" "oss"
teleport version
```

Bước 2:
Sau khi cài đặt thành công, port của teleport mặc định là 3080. Nếu không thấy có port 3080, hãy kiểm tra lại file /etc/teleport.yaml. Phần proxy_service phải bật enabled: yes và web_listen_addr: 0.0.0.0:3080
```
teleport configure > /etc/teleport.yaml
systemctl daemon-reload
systemctl enable teleport
systemctl start teleport
```

Cấu hình nginx, teleport luôn chạy https để bảo vệ dữ liệu. Nginc đóng vái trò là "SSL Termination" nhận HTTPS từ khách, rồi chuyển HTTPS tiếp tới teleport
```
location / {
    # 1. Đổi http thành https
    proxy_pass https://127.0.0.1:3080;

    # 2. Cấu hình WebSocket (Bắt buộc cho Teleport)
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # 3. Chuyển tiếp các Header quan trọng
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;

    # 4. Tắt kiểm tra SSL vì Teleport dùng cert nội bộ
    proxy_ssl_verify off; # Vì chứng chỉ mà Teleport tự sinh ra không được cấp bởi CA công khai, Nginx sẽ từ chối kết nối nếu không có dòng này
    proxy_ssl_server_name on;
}
```

Cấu hình đổi link hostname nội bộ sang domain
```
vi /etc/teleport.yaml

proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:3080
  # Thêm dòng này vào
  public_addr: teleport.codeinet.com:443
  # Nếu bạn muốn dùng cho cả SSH qua domain
  ssh_public_addr: teleport.codeinet.com:3023

sudo systemctl restart teleport
```

Bước 3: Tạo tài khoản Admin
Trong teleport, quyền hạn được quản lý thông qua các Roles:
- access: cho phép truy cập vào các tài nguyên SSH, Web UI
- editor: quyền chỉnh sửa cấu hình cụm
- auditor: chỉ xem được

Tạo tài khoản Admin
```
sudo tctl users add admin --roles=editor,access --logins=root,anhln

kết quả trả ra
User "admin" has been created but requires a password. Share this URL with the user to complete user setup, link is valid for 1h:
https://teleport.codeinet.com:443/web/invite/xxxxxx
```

