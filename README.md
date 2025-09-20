# Dự án Cloud Hybrid Proxmox + EC2 Reverse Proxy  

## Mục tiêu dự án  
- Biến server cũ tại nhà thành **lab ảo hóa** với Proxmox VE 7.4.  
- Chạy song song nhiều VM (Ubuntu, Windows Server).  
- Truy cập VM từ bất kỳ đâu qua **AWS EC2 Hong Kong** acting như jump host / reverse proxy.  
- Thực hành chia tài nguyên, quản lý mạng, tunnel, và fix lỗi thực tế.  

---

## Kiến trúc phần cứng & chia tài nguyên  

**Server tại nhà**:  
- CPU: Intel i3-4130 (2C/4T).  
- RAM: 8GB DDR3.  
- Disk: 250GB SSD.  

**Phân bổ VM trong Proxmox**:  
- **Ubuntu VM**: 1 cores (1 thread), 1.5GB RAM, 40GB disk → dùng SSH, test service.  
- **Windows Server VM**: 1 cores (2 threads), 4GB RAM, 100GB disk → bật Remote Desktop (RDP).  
- **Host Proxmox** giữ lại ~1GB RAM.  

---

## Các bước triển khai  

1. Cài Proxmox VE (fix boot ISO → cài bằng USB, chỉnh GRUB).  
2. Tạo VM Ubuntu + Windows.  
3. Bật SSH (Ubuntu), Remote Desktop (Windows).  
4. Cấu hình autossh reverse tunnel từ Proxmox → EC2:  

   ```bash
    autossh -M 0 -N -f \
    -o ServerAliveInterval=60 -o ServerAliveCountMax=3 \
    -i ~/.ssh/ec2-hk.pem   \   
    -R 0.0.0.0:8006:127.0.0.1:8006  \    
    -R 0.0.0.0:2222:192.168.32.180:22   \
    -R 0.0.0.0:33890:192.168.32.139:3389   \
    ubuntu@<EC2_PUBLIC_IP>
   ```

5. Chỉnh `sshd_config` trên EC2:  
   ```
   AllowTcpForwarding yes
   GatewayPorts clientspecified
   ```  

6. Mở Security Group AWS inbound: 8006, 2222, 33890.  
7. Tạo systemd service cho autossh (auto reconnect).  

```
[Unit]
Description=Reverse SSH Tunnel to EC2 HK
After=network-online.target
Wants=network-online.target

[Service]
User=root
ExecStart=/usr/bin/autossh -M 0 -N \
  -o ServerAliveInterval=60 -o ServerAliveCountMax=3 \
  -i /root/.ssh/ec2-hk.pem \
  -R 0.0.0.0:8006:127.0.0.1:8006 \
  -R 0.0.0.0:2222:192.168.32.180:22 \
  -R 0.0.0.0:33890:192.168.32.139:3389 \
  ubuntu@<EC2_PUBLIC_IP>
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## Những sự cố & cách fix  

- **ISO Proxmox không boot** → tạo USB boot.  
- **Netplan mất config sau reboot** → tạo `00-installer-config.yaml`.  
- **SSH root deny** → chỉnh `PermitRootLogin` hoặc tạo user mới.  
- **Tunnel chỉ listen 127.0.0.1** → bật `GatewayPorts clientspecified`.  
- **Port forwarding failed** → kill autossh cũ, restart.  
- **Security Group chặn** → mở đúng port inbound.  
- **RDP không vào** → bật Remote Desktop + mở firewall trong Windows.  
- **Swap nhiều** → giảm RAM Ubuntu xuống 2GB, Win 4GB.  

---

## Kỹ năng đã học  

- Quản lý VM, chia CPU/RAM trong Proxmox.  
- Networking: NAT, port forward, netplan, firewall, AWS SG.  
- Reverse SSH tunnel với autossh.  
- systemd service để tự động hóa.  
- Quản lý sshd_config, authorized_keys, ufw.  
- Cấu hình RDP & firewall Windows.  
- Thói quen fix lỗi thực tế từ OS đến cloud.  

---

## Mở rộng  
- Cài **WireGuard VPN** để VM ở nhà ra Internet bằng IP EC2 (Hong Kong).  
- Public thêm HTTP/HTTPS service từ VM qua EC2.  

---

## Kiến trúc minh họa  
```text
   [ Internet ]
        │
   [ AWS EC2 Hong Kong ]
      (Jump Host / Reverse Proxy)
        │
   ┌────┴─────────────┐
   │                  │
[Proxmox UI]     [VMs tại nhà]
  :8006         Ubuntu VM :2222
                 Win VM   :33890
```