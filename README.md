# Xây Dựng Mô Hình Quản Trị Mạng Với Zabbix Trên EVE-NG

Mô phỏng hệ thống mạng doanh nghiệp gồm tường lửa, core, distribution và access switch — giám sát tập trung bằng Zabbix 7.

---

## Tổng Quan

| Thành phần          | Chi tiết               |
|---------------------|------------------------|
| Tường lửa           | 1 x FortiGate          |
| Core Switch         | 1 x IOSvL2             |
| Distribution Switch | 2 x IOSvL2             |
| Access Switch       | 2 x IOSvL2             |
| Monitoring Server   | Zabbix Appliance 7.4.8 |
| Client giám sát     | Lubuntu 24.04          |  

### VLANs

| VLAN | Tên        | Subnet       |
|------|------------|--------------|
| 10   | Monitor    | 10.0.10.0/24 |
| 20   | Staff      | 10.0.20.0/24 |
| 99   | Management | 10.0.99.0/24 |

---

## Topology

<img width="825" height="744" alt="Screenshot 2026-03-24 052232" src="https://github.com/user-attachments/assets/43901a4f-c483-4795-a7c8-0d13653caa34" />

---

## Môi Trường và Công Cụ

- Host OS: Windows 11
- Hypervisor: VMware Workstation
- Lab: EVE-NG Community
- File Transfer: WinSCP
- Remote Desktop: UltraVNC
- Terminal: SecureCRT, PuTTY

---

## Images Sử Dụng

| Image                    | Vai trò                     | Phiên bản |
|--------------------------|-----------------------------|-----------|
| `lubuntu`                | Máy monitor (client Zabbix) | 24.04     |
| `zabbix_appliance`       | Zabbix Server               | 7.4.8     |
| `fortinet-FGT`           | Tường lửa (FortiGate)       | v7.2.4.F  |
| `viosl2-adventerprisek9` | Core / Dis / Access switch  | 20200929  |

Các image có thể tải trên mạng hoặc lấy từ link: 
[Eve-ng GNS3 Image](https://github.com/hegdepavankumar/Cisco-Images-for-GNS3-and-EVE-NG)

---

## Cài Đặt Môi Trường

### 1. Tạo máy ảo EVE-NG trong VMware

- RAM: 16 GB (có thể giảm tùy nhu cầu)
- Disk: 80 GB
- Yêu cầu bật Intel VT-x hoặc AMD-V trong BIOS

### 2. Truy cập EVE-NG

Sau khi tạo máy ảo, truy cập IP của EVE-NG trên trình duyệt và đăng nhập:

- Username: `admin`
- Password: `eve`

Sau đó tạo lab mới và bắt đầu thêm thiết bị.

### 3. Cài EVE-NG Client Pack

Cài đặt EVE-NG Windows Client Pack để tích hợp sẵn các công cụ: UltraVNC, PuTTY, SecureCRT.

---

## Thêm Images Vào EVE-NG

1. Dùng WinSCP chuyển image vào thư mục: /opt/unetlab/addons/qemu/

2. Đặt đúng định dạng tên thư mục theo chuẩn EVE-NG.

   <img width="744" height="329" alt="Screenshot 2026-03-24 053045" src="https://github.com/user-attachments/assets/a847c9fc-3523-4ac1-9c89-b57be7de490c" />

3. Chạy lệnh sau để hệ thống nhận diện image: /opt/unetlab/wrappers/unl_wrapper -a fixpermissions

Tham khảo thêm: [EVE-NG HowTo Documentation](https://www.eve-ng.net/index.php/documentation/howtos/)

---

## Cấu Hình Thiết Bị

### Tường Lửa (FortiGate)

| Interface | Vai trò | Cấu hình                               |
|-----------|---------|----------------------------------------|
| port1     | WAN     | Nối Cloud net, bật DHCP, cho phép HTTP |
| port2     | LAN     | 10.0.0.1/30, nối xuống Core            |

- NAT: Tạo Firewall Policy NAT traffic từ port2 sang port1

  <img width="1919" height="391" alt="image" src="https://github.com/user-attachments/assets/db7c28f4-c065-4cbb-88c4-3d88c0024cbe" />
  
- Static Route: Thêm route vào mạng nội bộ qua Core

  <img width="1917" height="430" alt="image" src="https://github.com/user-attachments/assets/edcc53f9-a953-4165-b1aa-9babfee8e2b0" />

---

### Core Switch

- Bật IP Routing
- Cấu hình các interface vật lý dạng no switchport
- Cấu hình IP Loopback làm Router-ID và SNMP
- Bật OSPF
- Thêm default route lên tường lửa

  <img width="845" height="255" alt="image" src="https://github.com/user-attachments/assets/01425fa4-be37-4ff4-a424-514c95c4e1d3" />

---

### Distribution Switch (x2)

- no switchport ở các port nối lên Core
- Trunk ở các port nối xuống Access
- Port-channel Trunk ở kết nối giữa 2 Dis switch
- Tạo các VLAN (10, 20, 99)
- Cấu hình SVI + HSRP giữa 2 Dis switch
- Cấu hình DHCP Server cho các VLAN
- Cấu hình IP Loopback làm Router-ID và SNMP
- Bật OSPF
- Thêm default route lên Core

  <img width="833" height="439" alt="image" src="https://github.com/user-attachments/assets/dae9fe7f-dcb1-4d0f-85f0-1ce4c668f2db" />

---

### Access Switch (x2)

- no ip routing
- Tạo các VLAN (10, 20, 99)
- Trunk ở port nối lên Dis
- Access ở port nối tới các máy đầu cuối
- Cấu hình SVI VLAN 99 cho SNMP
- Cấu hình ip default-gateway trỏ về Dis

  <img width="826" height="326" alt="image" src="https://github.com/user-attachments/assets/45ea1a16-5038-4b3f-962c-ebdb9e8450bf" />

---

## Quản Trị Mạng Với Zabbix

### Bật SNMP trên thiết bị

- FortiGate: Bật SNMP v2c, community `public`, cho phép SNMP trên port2
  
  <img width="1919" height="862" alt="image" src="https://github.com/user-attachments/assets/c331ab84-9360-4d56-82ee-f664bca9fc49" />

- Core / Dis / Access Switch: Bật SNMP bằng lệnh `snmp-server community public RO`
  
- LubuntuCài SNMP agent, cho phép mọi kết nối UDP:161

  <img width="807" height="424" alt="image" src="https://github.com/user-attachments/assets/5daa8ac4-8cd6-4857-bae2-e69c17b01800" />

### Cài đặt Zabbix Appliance

Cài đặt IP tĩnh cho Zabbix server trong file /etc/sysconfig/network-scripts/ifcfg-eth0:

<img width="516" height="272" alt="Screenshot 2026-03-23 061603" src="https://github.com/user-attachments/assets/b7829576-921f-4f8f-93da-93174068bffd" />

### Thêm Host Vào Zabbix

1. Truy cập IP của Zabbix Server trên trình duyệt từ máy Lubuntu
2. Đăng nhập: Username `Admin`, Password `zabbix`
3. Vào Configuration > Hosts > Create Host
4. Chọn Template phù hợp (Zabbix có sẵn template cho FortiGate, Cisco, Linux,...)
5. Chọn Interface: SNMP, điền IP thiết bị, port 161, community `public`

<img width="1024" height="707" alt="image" src="https://github.com/user-attachments/assets/50913fa3-a301-40f3-97b6-48199f488aed" />

<img width="1025" height="668" alt="image" src="https://github.com/user-attachments/assets/a3a05721-b12e-46bc-8468-3354bdab27ed" />

### Tùy Chỉnh Dashboard

- Clone template có sẵn hoặc tạo Custom Template
- Thêm Items mới để theo dõi các metric cần thiết.

  <img width="1024" height="702" alt="image" src="https://github.com/user-attachments/assets/8a31a4dc-3c47-4dff-9ab6-ab1753f9b247" />

- Tạo Widgets tùy chỉnh cho Dashboard

  <img width="1026" height="728" alt="image" src="https://github.com/user-attachments/assets/67744c33-008c-4a21-9290-536fa355668d" />

---

## Testing
- Test DHCP
  
  <img width="911" height="375" alt="image" src="https://github.com/user-attachments/assets/ca13ff82-fe59-4177-9eb8-97c4852a2000" />

- Test Internet

  <img width="655" height="251" alt="image" src="https://github.com/user-attachments/assets/931bdfeb-f39c-43ac-9620-a5afe6dc5e75" />
  
- Dashboard Zabbix

  <img width="1028" height="743" alt="image" src="https://github.com/user-attachments/assets/bea90285-1e50-423f-a4fc-4342f69e5591" />


## Troubleshooting

### EVE-NG không chạy được do xung đột VT-x

- EVE-NG yêu cầu Intel VT-x hoặc AMD-V. Trên một số máy Windows 11, tính năng này xung đột với Credential Guard / Virtual Based Security. Tắt Credential Guard theo hướng dẫn tại: [Turn off VBS](https://community.broadcom.com/vmware-cloud-foundation/discussion/windows-11-24h2-hsot-how-to-disable-virtual-based-security)

### Cấu hình FortiGate chưa có hiệu lực

- Một số cấu hình trên FortiGate cần tắt rồi bật lại interface hoặc restart thiết bị mới áp dụng.

### Host mới trên Zabbix hiển thị "Unknown"

- Dù SNMP đã hoạt động, host mới tạo vẫn có thể hiển thị trạng thái Unknown. Giải pháp: Tắt và bật lại Zabbix Appliance (restart VM).

### Một số item SNMP không trả về kết quả
 
- Khi giám sát qua SNMP, một số item có thể không lấy được dữ liệu dù kết nối SNMP hoạt động bình thường. Nguyên nhân có thể do OID trong template khác với OID thiết bị ảo hỗ trợ. Ví dụ ở phần tùy chỉnh dashboard, template có sẵn OID cho bộ nhớ còn trống nhưng không trả về kết quả, cần thay bằng OID khác phù hợp hơn. Tham khảo danh sách OID tại: [SNMP OIDs for Switch Monitoring](https://www.10-strike.com/network-monitor/pro/useful-snmp-oids.shtml)

---
