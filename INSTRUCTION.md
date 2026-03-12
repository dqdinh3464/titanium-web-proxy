# Titanium Web Proxy — Hướng dẫn & Cách hoạt động

## 1. Tổng quan

Titanium Web Proxy là một **HTTP/HTTPS proxy server** chạy trên máy tính local. Nó đóng vai trò **người trung gian (Man-in-the-Middle)** giữa thiết bị Android và internet, cho phép chuyển tiếp, theo dõi và can thiệp vào traffic mạng.

## 2. Kiến trúc tổng quan

```
┌──────────────┐       ┌──────────────────────┐       ┌──────────────┐
│  Android     │──────▶│  Titanium Web Proxy   │──────▶│  External    │
│  Device      │       │  (chạy trên PC local) │       │  Proxy       │
│  (qua ADB)   │◀──────│                      │◀──────│  (9Proxy)    │
└──────────────┘       └──────────────────────┘       └──────────────┘
```

Mô hình 2 tầng proxy:
- **Tầng 1 (Local)**: Titanium Web Proxy chạy trên PC, lắng nghe kết nối từ thiết bị Android
- **Tầng 2 (External)**: 9Proxy (hoặc proxy bên ngoài khác) cung cấp IP khác nhau cho mỗi thiết bị

## 3. Cách sử dụng trong dự án

Trong `adbphonenetcore`, lớp `ProxyServer` (`Helper/ProxyServer.cs`) bọc (wrap) Titanium Web Proxy với các bước sau:

### Bước 1: Khởi tạo Proxy Server

```csharp
proxyServer = new Titanium.Web.Proxy.ProxyServer();
proxyServer.ForwardToUpstreamGateway = true;
```

- Tạo instance proxy server mới
- Bật chế độ **forward traffic** lên upstream gateway (proxy bên ngoài)

### Bước 2: Quản lý chứng chỉ SSL (HTTPS)

```csharp
if (!IsTitaniumCertificateInstalled())
    proxyServer.CertificateManager.CreateRootCertificate();
proxyServer.CertificateManager.TrustRootCertificate();
```

- Kiểm tra xem **Root CA certificate** của Titanium đã được cài vào máy chưa
- Nếu chưa → tạo Root CA mới
- Trust Root CA để hệ thống tin tưởng các certificate do proxy tạo ra
- Đây là chìa khóa để proxy có thể **giải mã HTTPS** (Man-in-the-Middle): khi thiết bị Android kết nối đến trang HTTPS qua proxy, Titanium sẽ **tự động tạo certificate giả** cho domain đó

### Bước 3: Thiết lập External Proxy (Upstream)

```csharp
proxyServer.UpStreamHttpProxy = ExternalProxy;   // ví dụ: 9Proxy
proxyServer.UpStreamHttpsProxy = ExternalProxy;
```

- Cấu hình proxy bên ngoài (ví dụ: 9Proxy) làm **upstream**
- Mọi traffic sẽ đi theo chuỗi: `Android → Titanium (local) → 9Proxy (external) → Internet`
- Mỗi thiết bị Android có thể dùng một **IP khác nhau** thông qua 9Proxy

### Bước 4: Tạo Endpoint lắng nghe

Hỗ trợ 2 loại endpoint:

**HTTP Proxy:**
```csharp
var httpEndPoint = new ExplicitProxyEndPoint(IPAddress.Any, HttpPort, false);
proxyServer.AddEndPoint(httpEndPoint);
```

**SOCKS Proxy:**
```csharp
var socksEndPoint = new SocksProxyEndPoint(IPAddress.Any, SocksPort, false);
proxyServer.AddEndPoint(socksEndPoint);
```

- Mở port trên PC để lắng nghe kết nối từ thiết bị Android
- Mỗi thiết bị Android được ADB forward đến một port riêng
- Tham số `false` = không bật SSL decryption mặc định trên endpoint

### Bước 5: Khởi động

```csharp
proxyServer.Start();
```

## 4. Luồng xử lý Request

```
1. Ứng dụng trên Android gửi HTTP/HTTPS request
       ↓
2. ADB reverse forward chuyển request → localhost:{port} trên PC
       ↓
3. Titanium Web Proxy nhận request tại endpoint đã cấu hình
       ↓
4. Nếu là HTTPS → giải mã SSL bằng fake certificate (MITM)
       ↓
5. Forward request đến External Proxy (9Proxy)
       ↓
6. 9Proxy gửi request ra internet với IP proxy được cấp
       ↓
7. Response trả về theo chiều ngược lại:
   Internet → 9Proxy → Titanium → ADB forward → Android
```

## 5. Tại sao cần Titanium thay vì kết nối trực tiếp đến 9Proxy?

| Lý do | Giải thích |
|-------|-----------|
| **Quản lý port** | Mỗi thiết bị cần 1 proxy riêng. Titanium cho phép tạo nhiều local proxy endpoint, mỗi cái forward đến 1 external proxy khác nhau |
| **Chuyển đổi giao thức** | Linh hoạt chuyển đổi giữa HTTP proxy ↔ SOCKS proxy tùy nhu cầu |
| **Giải mã SSL** | Có thể đọc/sửa HTTPS traffic nếu cần (debug, inject headers, kiểm tra response) |
| **Quản lý tập trung** | Quản lý tất cả proxy connections từ một nơi, dễ start/stop/dispose |
| **Tự động chọn port** | Hàm `GetNextFreePort()` tự tìm port trống, tránh xung đột |

## 6. Vòng đời (Lifecycle)

```
StartProxyServer()
       ↓
[Proxy đang chạy — Android traffic đi qua]
       ↓
StopProxyServer() → Dispose()
       ↓
   ┌─ Xóa upstream proxy (UpStreamHttpProxy = null)
   ├─ Khôi phục proxy settings gốc của hệ thống
   ├─ Dừng proxy server
   └─ Giải phóng tài nguyên (Dispose)
```

## 7. Các tính năng chính của Titanium Web Proxy

- **Đa luồng, bất đồng bộ** — sử dụng connection pooling, certificate cache, buffer pooling
- **Chặn/sửa request & response** — xem, sửa đổi, chuyển hướng hoặc chặn traffic
- **SOCKS4/5** — hỗ trợ proxy SOCKS ngoài HTTP
- **Mutual SSL Authentication** — xác thực SSL hai chiều
- **Kerberos/NTLM** — hỗ trợ xác thực trên mạng Windows domain

## 8. Cấu trúc mã nguồn

```
/src/Titanium.Web.Proxy/
  ProxyServer.cs              — Lớp chính: quản lý endpoints, start/stop server
  RequestHandler.cs           — Xử lý request đến
  ResponseHandler.cs          — Xử lý response trả về
  ExplicitClientHandler.cs    — Xử lý HTTP proxy (explicit mode)
  SocksClientHandler.cs       — Xử lý SOCKS proxy
  TransparentClientHandler.cs — Xử lý transparent proxy (reverse proxy)
  /Certificates               — Quản lý SSL certificate (tạo, lưu cache, trust)
  /Compression                — Nén/giải nén traffic (gzip, deflate, brotli)
  /EventArguments             — Event args cho BeforeRequest, BeforeResponse, etc.
  /Handlers                   — Các handler xử lý kết nối
  /Http                       — HTTP parsing (headers, body, chunked encoding)
  /Http2                      — Hỗ trợ HTTP/2
  /Models                     — Data models (ExternalProxy, ProxyEndPoint, etc.)
  /Network                    — TCP connection management
  /ProxySocket                — Socket wrapper cho proxy connections
  /WebSocket                  — WebSocket proxy support
/examples                     — Ví dụ sử dụng (Console + WPF)
/tests                        — Unit tests
```

## 9. Lưu ý quan trọng

> ⚠️ Đây là thư viện **bên thứ 3 được fork/vendor** vào dự án. Không nên sửa đổi core logic trừ khi thật sự cần thiết. Ưu tiên sử dụng các bản vá từ upstream repo: [justcoding121/titanium-web-proxy](https://github.com/justcoding121/titanium-web-proxy)
