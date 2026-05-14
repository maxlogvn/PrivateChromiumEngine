# Chromium Anti-Detect Engine

Thư viện điều khiển trình duyệt Chromium chống bot detection, tích hợp fingerprint, proxy và quản lý profile đa phiên.

Được xây dựng trên nền `playwright-core` — tương thích hoàn toàn với Playwright API hiện có, không cần viết lại code nghiệp vụ.

---

## Cài đặt

```bash
npm install fingerprint-chromium-engine
```

> **Lưu ý:** Thư viện yêu cầu `playwright-core` là peer dependency.
> Nếu dự án chưa có, cần cài thêm:[microsoft/playwright](https://github.com/microsoft/playwright)
>

---

## Tính năng

### Fingerprint từ thiết bị thật, inject ở tầng C native

Không sinh fingerprint ngẫu nhiên — engine sử dụng fingerprint thu thập từ các thiết bị thực tế, sau đó inject trực tiếp vào Chromium ở cấp độ C/C++. Kết quả là mọi thuộc tính đều trả về giá trị native, không có dấu hiệu bị override dưới bất kỳ hình thức kiểm tra nào.

**Navigator & Platform**
- Navigator properties (thiết bị, trình duyệt, locale, OS...)
- Network headers `Accept-Language` và `User-Agent` tự động khớp với navigator
- Kích thước & độ phân giải màn hình, inner/outer viewport
- `devicePixelRatio` & HiDPI/Retina screen emulation

**Đồ họa**
- WebGL parameters, supported extensions, context attributes & shader precision formats
- Canvas 2D — thêm nhiễu chống canvas fingerprinting
- Font fingerprinting (hỗ trợ FontPack đồng bộ font hệ thống)

**Media & Hardware**
- AudioContext sample rate, output latency & max channel count
- Device voices & speech playback rates
- Số lượng microphone, webcam, speaker available
- Battery API, Sensor API (gia tốc kế, con quay hồi chuyển)
- ClientRects & DOM element coordinates

**Mạng & Vị trí**
- WebRTC IP spoofing ở tầng protocol — không thể bị detect qua JS
- Geolocation, timezone & locale

### Proxy & Môi trường thông minh

Không chỉ định tuyến traffic — engine tự động đồng bộ toàn bộ môi trường trình duyệt theo IP proxy:

- Timezone — múi giờ tự động khớp với vị trí địa lý của proxy
- Ngôn ngữ — `Accept-Language`, `navigator.language` theo quốc gia proxy
- Geolocation — vị trí địa lý theo IP (tuỳ chọn)
- WebRTC — che giấu hoặc thay thế IP rò rỉ qua WebRTC
- DNS tùy chỉnh — hỗ trợ `custom-proxy` và `custom-direct` để tránh DNS leak
- QUIC — tùy chọn bật giao thức QUIC nếu proxy hỗ trợ UDP

### Profile đa phiên

Duy trì trạng thái đăng nhập, cookies, localStorage và lịch sử giữa các lần chạy:

- Mỗi profile độc lập, được lưu theo đường dẫn tùy chọn
- Tự động khôi phục fingerprint và proxy đã dùng ở phiên trước
- Lưu profile khi đóng — có thể chỉ định đường dẫn lưu khác nhau mỗi lần

### Tương thích Playwright 100%

Trả về `BrowserContext` chuẩn của `playwright-core`. Toàn bộ API Playwright (`page`, `locator`, `expect`, `route`...) hoạt động bình thường — không cần thay đổi code nghiệp vụ.

---

## Bắt đầu nhanh

```ts
import { Chromium } from 'fingerprint-chromium-engine';

const context = await Chromium
  .useFingerprint(fingerprintData)
  .useProxy('http://user:pass@127.0.0.1:8080')
  .useProfile('./profiles/user_01')
  .launch({ headless: false })
  .newContext();

const page = await context.newPage();
await page.goto('https://example.com');

await Chromium.quit();
```

> Tất cả method cấu hình (`use*`) trả về `this` — hỗ trợ method chaining.
> Bắt buộc gọi trước `launch()`. Sau khi `launch()`, cấu hình bị khóa.

---

## Hướng dẫn sử dụng

### Fingerprint

```ts
Chromium.useFingerprint(fingerprintData, {
  usePerfectCanvas: true,   // Canvas chính xác theo fingerprint
  safeWebGL: true,          // Che giấu GPU renderer & vendor
  safeAudio: true,          // Che giấu thông tin audio hardware
  useFontPack: true,        // Đồng bộ font với fingerprint mục tiêu
})
```

> `useFontPack` yêu cầu cài đặt [FontPack từ Bablosoft](https://wiki.bablosoft.com/doku.php?id=fontpack). Nếu chưa cài, engine tự fallback.

| Tùy chọn | Mô tả | Mặc định |
|---|---|---|
| `emulateDeviceScaleFactor` | Giả lập màn hình HiDPI/Retina. Render ở độ phân giải cao hơn, tốn thêm tài nguyên. Các giá trị JS như `devicePixelRatio` luôn được thay thế đúng dù bật hay tắt | `true` |
| `emulateSensorAPI` | Giả lập Sensor API (gia tốc kế, con quay hồi chuyển...). Nên bật khi giả lập fingerprint thiết bị di động | `true` |
| `usePerfectCanvas` | Thay thế dữ liệu Canvas chính xác theo fingerprint. Yêu cầu fingerprint phải chứa dữ liệu PerfectCanvas | `true` |
| `useFontPack` | Đồng bộ font hệ thống theo fingerprint, tránh sai lệch khi fingerprint mục tiêu có nhiều font hơn máy hiện tại | `true` |
| `safeElementSize` | Che giấu tọa độ thực của DOM element, chống ClientRects fingerprinting | `false` |
| `safeBattery` | Giả lập Battery API với giá trị khác nhau mỗi phiên. Trả về 100% nếu thiết bị gốc không có Battery API | `true` |
| `safeCanvas` | Thêm nhiễu vào Canvas 2D để chống canvas fingerprinting | `true` |
| `safeAudio` | Thêm nhiễu vào Web Audio API, che giấu sample rate và số kênh âm thanh | `true` |
| `safeWebGL` | Thêm nhiễu vào WebGL, che giấu tên GPU renderer & vendor | `true` |

---

### Proxy

```ts
Chromium.useProxy('http://user:pass@127.0.0.1:8080', {
  changeBrowserLanguage: true, // Đồng bộ ngôn ngữ trình duyệt theo quốc gia proxy
  changeTimezone: true,        // Đồng bộ múi giờ theo IP proxy
  changeGeolocation: false,    // Đồng bộ vị trí địa lý (mặc định tắt)
  changeWebRTC: 'replace',     // Thay IP WebRTC bằng IP proxy
})
```

**Tùy chọn cơ bản:**

| Tùy chọn | Mô tả | Mặc định |
|---|---|---|
| `changeBrowserLanguage` | Đổi `Accept-Language` và `navigator.language` theo quốc gia proxy | `true` |
| `changeTimezone` | Đổi múi giờ trình duyệt theo IP proxy | `true` |
| `changeGeolocation` | Đổi geolocation theo IP proxy. Nếu tắt, trình duyệt từ chối mọi yêu cầu truy cập vị trí | `false` |
| `enableTunneling` | Bật/tắt hệ thống tunneling tích hợp. Tắt khi đã có VPN hoặc muốn kết nối trực tiếp | `true` |
| `enableQUIC` | Bật giao thức QUIC (UDP). Chỉ bật nếu proxy server hỗ trợ UDP | `false` |

**Tùy chọn WebRTC:**

| Giá trị `changeWebRTC` | Hành vi |
|---|---|
| `enable` | Bật WebRTC — lộ IP thật |
| `disable` | Tắt hoàn toàn WebRTC |
| `replace` | Thay IP WebRTC bằng IP proxy (khuyến nghị) |

Khi dùng `changeWebRTC: 'replace'`, có thể kiểm soát chi tiết IP hiển thị qua WebRTC:

```ts
Chromium.useProxy('http://...', {
  changeWebRTC: 'replace',
  publicIPv4: 'auto',     // tự lấy IPv4 công khai từ proxy
  publicIPv6: 'disable',  // ẩn IPv6 công khai
  privateIPv4: 'local',   // dùng IP nội bộ thực của máy
  privateIPv6: 'disable', // ẩn IPv6 nội bộ
})
```

| Tùy chọn | Mô tả | Mặc định |
|---|---|---|
| `publicIPv4` | IPv4 công khai hiển thị qua WebRTC. Nhận `'auto'`, `'disable'` hoặc IP cụ thể | `'auto'` |
| `publicIPv6` | IPv6 công khai hiển thị qua WebRTC. Nhận `'auto'`, `'disable'` hoặc IP cụ thể | `'auto'` |
| `privateIPv4` | IPv4 nội bộ hiển thị qua WebRTC. Nhận `'local'`, `'disable'`, IP cụ thể hoặc dải `'private class a/b/c'` | `'local'` |
| `privateIPv6` | IPv6 nội bộ hiển thị qua WebRTC. Nhận `'local'`, `'disable'`, IP cụ thể hoặc `'unique local address'` | `'local'` |

**Tùy chọn tra cứu IP:**

| Tùy chọn | Mô tả | Mặc định |
|---|---|---|
| `detectExternalIP` | Tự động phát hiện IP công khai thực qua proxy. Hữu ích khi IP kết nối proxy khác IP hiển thị ra ngoài. Có thể cấu hình riêng cho IPv4/IPv6 | `true` |
| `ipInfoMethod` | Phương thức tra cứu thông tin địa lý từ IP. `'database'` — nội bộ, nhanh nhưng kém chính xác. `'ip-api.com'` — bên ngoài, chính xác hơn nhưng giới hạn 45 request/phút/IP (bản free) | `'database'` |
| `ipInfoKey` | API key của [ip-api.com](https://ip-api.com/) bản trả phí. Chỉ có hiệu lực khi `ipInfoMethod` là `'ip-api.com'` | `''` |

**Tùy chọn nguồn lấy IP tùy chỉnh:**

Dùng khi cần lấy IP công khai từ một service riêng thay vì service mặc định:

```ts
Chromium.useProxy('http://...', {
  ipExtractionURL: 'https://api.myservice.com/ip',
  ipExtractionMethod: 'jsonpath',
  ipExtractionParam: '$.ip',
})
```

Có thể cấu hình riêng cho IPv4 và IPv6:

```ts
Chromium.useProxy('http://...', {
  ipExtractionURL: { v4: 'https://ipv4.service.com', v6: 'https://ipv6.service.com' },
  ipExtractionMethod: { v4: 'raw', v6: 'jsonpath' },
  ipExtractionParam: { v4: '', v6: '$.ip' },
})
```

| Tùy chọn | Mô tả | Mặc định |
|---|---|---|
| `ipExtractionURL` | URL trả về địa chỉ IP công khai hiện tại qua proxy | `''` |
| `ipExtractionMethod` | Phương thức trích xuất IP từ response: `'raw'`, `'jsonpath'`, `'xpath'`, `'regexp'` | `'raw'` |
| `ipExtractionParam` | Tham số dùng để trích xuất, kết hợp với `ipExtractionMethod` | `''` |

**Tùy chọn DNS:**

```ts
Chromium.useProxy('http://...', {
  dnsMode: 'custom-direct', // phân giải DNS cục bộ, traffic còn lại qua proxy
  dnsIP: '1.1.1.1',
})
```

| Giá trị `dnsMode` | Hành vi |
|---|---|
| `system-proxy` | Dùng DNS hệ thống, hostname được gửi đến proxy để phân giải |
| `custom-proxy` | Dùng DNS tùy chỉnh của Chrome, truy vấn DNS qua proxy (proxy phải hỗ trợ UDP) |
| `custom-direct` | Dùng DNS tùy chỉnh của Chrome, phân giải DNS cục bộ, traffic còn lại qua proxy (khuyến nghị) |

> Khi dùng `custom-proxy` hoặc `custom-direct`, cần chỉ định `dnsIP`. Mặc định `dnsIP` là `'1.1.1.1'`.
> `custom-proxy` yêu cầu proxy hỗ trợ UDP. Nếu proxy chỉ hỗ trợ TCP, dùng `custom-direct` hoặc `system-proxy`.

---

### Profile

```ts
// Lần đầu — tạo mới profile
Chromium.useProfile('./profiles/user_01')

// Các lần sau — tự động khôi phục session, proxy, fingerprint
Chromium.useProfile('./profiles/user_01', {
  loadProxy: true,       // khôi phục proxy từ phiên trước
  loadFingerprint: true, // khôi phục fingerprint từ phiên trước
})
```

Profile tự động lưu khi gọi `quit()`. Có thể ghi đè đường dẫn lưu:

```ts
await Chromium.quit('./profiles/user_01_backup');
```

---

### Thay thế Chromium launcher

Engine đi kèm một bản Chromium đã được patch sẵn để chống detection. Trong trường hợp đặc biệt, có thể thay thế bằng launcher tùy chỉnh:

```ts
Chromium.repackChromium(customLauncher)
```

> **Cảnh báo:** Chỉ dùng `repackChromium()` khi có lý do đặc biệt và hiểu rõ rủi ro. Launcher tùy chỉnh có thể không được patch đầy đủ, dẫn đến tăng nguy cơ bị detect.

---

### Vòng đời trình duyệt

```ts
// 1. Cấu hình (có thể chaining)
Chromium
  .useFingerprint(data)
  .useProxy('http://...')
  .useProfile('./profiles/user_01')

// 2. Khởi tạo engine — chỉ gọi một lần
  .launch({ headless: false })

// 3. Mở phiên duyệt
const context = await Chromium.newContext();
const page = await context.newPage();

// 4. Đóng và lưu
await Chromium.quit();
```

> `launch()` chỉ được gọi một lần. Gọi lại sẽ ném lỗi.
> `newContext()` chỉ cho phép một context tại một thời điểm. Gọi `quit()` trước khi tạo context mới.

---

## Lưu ý

**Giá trị mặc định `changeGeolocation`** — mặc định là `false`. Khi tắt, trình duyệt sẽ từ chối mọi yêu cầu truy cập vị trí từ trang web.

**IP Geolocation với ip-api.com** — bản free giới hạn 45 request/phút/IP. Vượt quá giới hạn nhận HTTP 429. Cân nhắc dùng bản Pro với `ipInfoKey` hoặc chuyển về `ipInfoMethod: 'database'` khi scale lớn.

**FontPack** — cần tải và cài đặt riêng từ [Bablosoft Wiki](https://wiki.bablosoft.com/doku.php?id=fontpack) để `useFontPack` hoạt động đúng.

**Thứ tự gọi** — `use*` -> `launch()` -> `newContext()` -> `quit()`. Sai thứ tự sẽ ném lỗi có mô tả rõ ràng.

---

## Đóng góp & Hỗ trợ

Gặp vấn đề hoặc muốn đề xuất tính năng — tạo Issue hoặc Pull Request tại:

https://github.com/maxlogvn/PrivateChromiumEngine/issues