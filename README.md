# CÀI ĐẶT VÀ CẤU HÌNH MÁY CHỦ VPN STRONGSWAN GATEWAY TRÊN UBUNTU 20.04

StrongSwan là một công cụ mã nguồn mở hoạt động như một trình nền khóa và sử dụng các giao thức Trao đổi Khóa Internet (IKEv1 và IKEv2) để bảo mật các kết nối giữa hai máy chủ. Bằng cách này, bạn có thể sử dụng StrongSwan để thiết lập Mạng riêng ảo (VPN). Các kết nối VPN từ máy khách đến máy chủ StrongSwan được mã hóa và cung cấp cổng an toàn cho các tài nguyên khác có sẵn trên máy chủ và mạng của nó. Hướng dẫn này chỉ cho bạn cách cài đặt và cấu hình máy chủ VPN cổng StrongSwan trên Ubuntu 20.04. Bạn cũng học cách thiết lập và kết nối với máy chủ StrongSwan từ máy khách Ubuntu, Windows và macOS.

## Step 1: Cài đặt StrongSwan

1. SSH vào máy chủ Ubuntu 20.04 của bạn.
2. Sử dụng APT để cài đặt StrongSwan và các plugin và thư viện hỗ trợ:

```bash
sudo apt install strongswan strongswan-pki libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins libtss2-tcti-tabrmd0 -y
```

## Step 2: Tạo khóa máy chủ và chứng chỉ

1. Sử dụng tiện ích dòng lệnh IPsec để tạo khóa riêng IPsec:

```bash
sudo ipsec pki --gen --size 4096 --type rsa --outform pem > /etc/ipsec.d/private/ca.key.pem
```

2. Tạo và ký chứng chỉ gốc với các cấu hình được bao gồm bên dưới. Đảm bảo thay thế giá trị của cấu hình CN bằng tên mong muốn cho máy chủ StrongSwan VPN:

```bash
ipsec pki --self --in /etc/ipsec.d/private/ca.key.pem --type rsa --dn "CN=<Name of this VPN Server>" --ca --lifetime 3650 --outform pem > /etc/ipsec.d/cacerts/ca.cert.pem
```

3. Tạo chứng chỉ riêng của máy chủ StrongSwan VPN:

```bash
ipsec pki --gen --size 4096 --type rsa --outform pem > /etc/ipsec.d/private/server.key.pem
```

4. Tạo chứng chỉ máy chủ lưu trữ. Có hai cách để tạo chứng chỉ, tuy nhiên, chúng không thể được trộn lẫn.

```bash
ipsec pki --pub --in /etc/ipsec.d/private/server.key.pem --type rsa | ipsec pki --issue --lifetime 3650 --cacert /etc/ipsec.d/cacerts/ca.cert.pem --cakey /etc/ipsec.d/private/ca.key.pem --dn "CN=<serverhost.ourdomain.tld>" --san="<server static IP address>" --outform pem > /etc/ipsec.d/certs/server.cert.pem
```

## Step 3: Cấu hình StrongSwan

1. Chỉnh sửa tệp /etc/sysctl.conf để cho phép chuyển tiếp gói cho IPsec và StrongSwan.

```bash
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
```

2. Định cấu hình tệp StrongSwan /etc/ipsec.conf.
```bash
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid=@server_domain_or_IP
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    right=%any
    rightid=%any
    rightauth=eap-mschapv2
    rightsourceip=10.10.10.0/24
    rightdns=8.8.8.8,8.8.4.4
    rightsendcert=never
    eap_identity=%identity
```
3. Tạo bí mật xác thực và truy cập trong /etc/ipsec.secrets.
```bash
: RSA "server-key.pem"
your_username : EAP "your_password"
```
## Step 4: Định cấu hình tường lửa
1. Cho phép kết nối SSH qua tường lửa để phiên hiện tại
```bash
sudo ufw allow OpenSSH
```
2. Kích hoạt tường lửa
```bash
sudo ufw enable
```
3. Cho phép lưu lượng UDP đến các cổng IPSec tiêu chuẩn, 500 và 4500:
```bash
sudo ufw allow 500,4500/udp
```

## Step 5: Kết nối với StrongSwan VPN
### Kết nối với StrongSwan VPN trên Windows 10
1. Nhấn WINDOWS+R để hiển thị hộp thoại Run, và nhập `mmc.exe` để khởi chạy Bảng điều khiển Quản lý Windows.
2. Từ menu Tệp, điều hướng đến Thêm hoặc Loại bỏ Snap-in, chọn Certificates từ danh sách các snap-ins có sẵn, và nhấn Add.
3. Chọn Computer Account và nhấn Next.
4. Chọn Local Computer, sau đó nhấn Finish.
5. Dưới nút Console Root, mở rộng mục Certificates (Local Computer), mở rộng Trusted Root Certification Authorities, và sau đó chọn mục Certificates.
6. Từ menu Action, chọn All Tasks và nhấn Import để hiển thị Wizard nhập chứng chỉ. Nhấn Next để bỏ qua phần giới thiệu.
7. Trên màn hình File to Import, nhấn nút Browse, đảm bảo rằng bạn thay đổi loại tệp từ "X.509 Certificate (.cer;.crt)" thành "All Files (.)", và chọn tệp `ca-cert.pem` mà bạn đã lưu. Sau đó nhấn Next.
8. Đảm bảo rằng Certificate Store được thiết lập thành Trusted Root Certification Authorities, và nhấn Next.
9. Nhấn Finish để nhập chứng chỉ.

### Kết nối với StrongSwan VPN trên MacOS
#### Theo các bước sau để nhập chứng chỉ:

1. Nhấp đúp vào tệp chứng chỉ. Keychain Access sẽ xuất hiện với một hộp thoại thông báo "Keychain Access is trying to modify the system keychain. Enter your password to allow this."
2. Nhập mật khẩu của bạn, sau đó nhấp vào Modify Keychain.
3. Nhấp đúp vào chứng chỉ VPN mới được nhập. Điều này mở ra một cửa sổ thuộc tính nhỏ nơi bạn có thể chỉ định mức độ tin cậy. Đặt IP Security (IPSec) thành Always Trust và bạn sẽ được nhắc nhập lại mật khẩu của mình. Cài đặt này tự động lưu sau khi nhập mật khẩu.

#### Bây giờ chứng chỉ đã được nhập và tin cậy, cấu hình kết nối VPN với các bước sau:

1. Đi tới System Preferences và chọn Network.
2. Nhấp vào nút "+" nhỏ ở phía dưới bên trái của danh sách các mạng.
3. Trong cửa sổ bật lên xuất hiện, đặt Interface thành VPN, đặt VPN Type thành IKEv2, và đặt tên cho kết nối.
4. Trong trường Server và Remote ID, nhập tên miền hoặc địa chỉ IP của máy chủ. Để trống Local ID.
5. Nhấp vào Authentication Settings, chọn Username và nhập tên người dùng và mật khẩu mà bạn đã định cấu hình cho người dùng VPN của mình. Sau đó nhấn OK.
6. Cuối cùng, nhấp vào Connect để kết nối với VPN. Bây giờ bạn nên đã kết nối thành công với VPN.

