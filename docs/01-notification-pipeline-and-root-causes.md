# HyperOS China Notification Reliability — Pipeline, nguyên nhân miss notification và hướng khắc phục

## 1. Mục tiêu tài liệu

Tài liệu này tổng hợp những gì đã nghiên cứu về hiện tượng **miss notification, trễ notification hoặc hoàn toàn không nhận notification** trên máy Xiaomi nội địa chạy China ROM, đặc biệt trong bối cảnh:

- Xiaomi nội địa / China ROM.
- HyperOS 4 beta.
- Android 17 / API 37.
- Không muốn root máy.
- Không muốn phụ thuộc Shizuku hoặc Developer options trong sử dụng hằng ngày.
- Ưu tiên độ tin cậy của notification hơn giao diện.
- Các app mục tiêu gồm nhóm dùng FCM như Facebook, Messenger, Instagram, Gmail, cùng các app ngân hàng và app quốc tế khác.

Mục tiêu cuối cùng không phải là "ép mọi app chạy nền vô hạn", mà là:

> Loại bỏ càng nhiều nguyên nhân phía thiết bị/Xiaomi càng tốt để notification đến đúng lúc, đồng thời không phá toàn bộ cơ chế quản lý pin của hệ thống.

### Yêu cầu sản phẩm cuối cùng

Sản phẩm phải là **một ứng dụng Android cài bằng APK**. Sau khi cài, toàn bộ trải nghiệm sử dụng phải diễn ra trong chính ứng dụng:

```text
cài APK
→ mở app
→ chạy kiểm tra / setup cần thiết
→ chọn app cần bảo vệ
→ bật protection
→ app tự giám sát, sửa và báo trạng thái
```

Không thiết kế sản phẩm cuối theo kiểu:

```text
cắm PC
→ chạy script ADB
→ nhập lệnh thủ công
→ mở nhiều tool rời
```

PC, ADB, JADX và các công cụ reverse-engineering chỉ phục vụ **quá trình phát triển và nghiên cứu**, không được là dependency khi người dùng sử dụng app.

Nếu Android/HyperOS bắt buộc cần quyền mà APK thường không thể tự cấp, có thể dùng **Shizuku hoặc ADB như bước bootstrap tạm thời**. Tuy nhiên mục tiêu kiến trúc là:

- Chỉ một APK của project cần được cài lâu dài.
- Không cần root.
- Không cần giữ Developer options bật trong sử dụng hằng ngày.
- Không cần giữ Shizuku chạy thường xuyên nếu quyền/policy có thể persist.
- Mọi chức năng có thể chạy không đặc quyền phải chạy trực tiếp trong APK.
- Chức năng nào thật sự cần đặc quyền phải được app phát hiện và giải thích rõ, không âm thầm phụ thuộc external tool.
- Sau reboot, nếu một state runtime bị reset thì người dùng chỉ cần mở app để audit/repair; không coi việc phải cắm lại PC là flow bình thường.

Không có giải pháp nào có thể bảo đảm 100% mọi notification luôn đến đúng giờ, vì còn phụ thuộc server của ứng dụng, FCM backend, mạng, token, priority của message và logic của chính ứng dụng. Nhưng có thể giảm rất mạnh phần lỗi do HyperOS China gây ra.

---

## 2. Pipeline notification tổng quát

Đối với phần lớn ứng dụng quốc tế dùng Firebase Cloud Messaging, đường đi đơn giản hóa như sau:

```text
App backend
    ↓
Firebase Cloud Messaging
    ↓
Google Play services
    ↓
FCM socket / transport
    ↓
Broadcast / delivery vào app đích
    ↓
App được phép chạy nền
    ↓
App xử lý payload
    ↓
NotificationManager / local notification
    ↓
Notification channel + permission
    ↓
Người dùng nhìn thấy notification
```

Điểm quan trọng là: **notification có thể chết ở bất kỳ tầng nào**.

Vì vậy hiện tượng "không có notification" không đồng nghĩa với "FCM không gửi". Có thể:

- FCM đã tới GMS nhưng GMS bị freeze sau đó.
- GMS nhận được nhưng Xiaomi chặn broadcast.
- Broadcast tới app nhưng app đang bị force-stop / hibernate / restricted.
- App xử lý message nhưng bị chặn tạo local notification.
- Notification được tạo nhưng channel hoặc permission bị tắt.

Một công cụ fix notification tốt phải phân biệt được các trường hợp này.

---

## 3. Các lớp Android gốc có thể ảnh hưởng notification

### 3.1 Doze

Android Doze được kích hoạt khi màn hình tắt, thiết bị không được sử dụng trong một khoảng thời gian và đạt điều kiện idle.

Trong Doze, Android có thể hạn chế:

- Network.
- JobScheduler.
- Sync.
- Alarm.
- Background execution.

FCM high-priority được thiết kế để vẫn có thể đánh thức thiết bị trong điều kiện phù hợp, nhưng normal-priority message có thể bị trì hoãn.

Doze bản thân không phải lỗi. Vấn đề xảy ra khi Xiaomi đặt thêm freezer và policy riêng chồng lên Doze.

### 3.2 App Standby / standby bucket

Ứng dụng ít được sử dụng có thể bị hạ xuống các standby bucket hạn chế hơn.

Các trạng thái này ảnh hưởng:

- Background jobs.
- Alarm.
- Background network.
- Khả năng được scheduler ưu tiên.

Đối với app cần notification tức thời, cần theo dõi bucket và tránh để policy vendor vô tình đẩy app critical vào trạng thái quá restrictive.

### 3.3 RUN_IN_BACKGROUND / RUN_ANY_IN_BACKGROUND

Đây là các AppOps liên quan khả năng hoạt động nền.

Nếu bị đặt về ignore/restricted, ứng dụng có thể nhận push không ổn định hoặc không có đủ thời gian xử lý message.

### 3.4 Cached App Freezer

Android hiện đại có cơ chế freeze process đang cached.

Cơ chế này thuộc AOSP và khác Xiaomi Greezer.

Không nên tắt toàn bộ Cached App Freezer chỉ để sửa notification vì:

- Đây là một phần thiết kế quản lý tài nguyên bình thường.
- Lifecycle event hợp lệ có thể thaw app.
- Disable global sẽ tăng tiêu thụ tài nguyên không cần thiết.

### 3.5 App hibernation

Ứng dụng bị hệ thống đưa vào trạng thái hibernate có thể mất khả năng nhận push bình thường.

Đây là một layer thường bị bỏ sót khi debug notification.

Các app critical cần được kiểm tra xem có:

- Bị auto revoke permission.
- Bị đánh dấu unused.
- Bị hibernated.

### 3.6 FLAG_STOPPED / force-stop state

Trên Android mới, trạng thái stopped được xử lý ngày càng chặt.

Một app ở trạng thái stopped có thể:

- Không nhận broadcast thông thường.
- Không được khởi động tự động bởi nhiều đường nền.
- Mất PendingIntent hoặc các đường wake-up cũ.

Vì vậy một app notification guard cần phân biệt:

- Process bị kill vì RAM.
- Process bị freeze.
- Package bị force-stop / stopped.

Ba trạng thái này không giống nhau.

### 3.7 LMKD

LMKD giết process khi có memory pressure.

Đây không phải cơ chế tiết kiệm pin của Xiaomi và không nên disable.

Một process bị LMKD kill có thể được hệ thống khởi động lại khi có event hợp lệ. Vì vậy mục tiêu là bảo đảm các đường wake-up còn hoạt động, không phải giữ mọi process sống vĩnh viễn.

---

## 4. Các lớp Xiaomi / HyperOS riêng

Đây là phần gây vấn đề lớn nhất trên China ROM.

## 4.1 PowerKeeper

PowerKeeper là thành phần quản lý pin và background policy của Xiaomi.

Nó có thể tham gia:

- Background restriction.
- Wakelock policy.
- Alarm policy.
- App lifecycle.
- No-restriction policy.
- GMS-specific behavior.
- Regenerate whitelist.

Vì PowerKeeper có private database và logic riêng, chỉ chỉnh vài setting Android chuẩn chưa chắc đủ.

---

## 4.2 `MILLET_NO_RESTRICT_APP`

Một setting quan trọng:

```text
Settings.System.MILLET_NO_RESTRICT_APP
```

Trên firmware đã được phân tích, Greezer và các policy Xiaomi đọc danh sách này để biết package nào thuộc nhóm không bị hạn chế theo một số đường freeze.

Vấn đề lớn:

- PowerKeeper coi setting này như một projection được tạo ra từ database riêng.
- Nếu chỉ dùng shell append `com.google.android.gms`, thay đổi đó không trở thành nguồn dữ liệu gốc của PowerKeeper.
- Khi người dùng đổi battery policy, package được thêm/xóa hoặc PowerKeeper reconcile state, nó có thể generate lại toàn bộ danh sách.
- Entry GMS được thêm thủ công có thể biến mất.

Do đó:

```text
write một lần
```

không đủ.

Cần:

```text
observe / poll
→ detect bị mất
→ merge lại
→ verify
```

Quan trọng: phải **merge**, không overwrite danh sách hiện tại vì người dùng/Xiaomi có thể đã có các package hợp lệ khác.

---

## 4.3 Greezer

Greezer là freezer riêng của Xiaomi.

Nó khác Cached App Freezer của Android.

Các firmware HyperOS đã cho thấy nhiều đường freeze khác nhau:

- GMS limiter.
- Aurogon.
- PowerStrategyMode.
- Nighttime freeze.
- Các policy theo whitelist.

Nếu GMS bị freeze:

```text
com.google.android.gms
    ↓ frozen
FCM TCP socket chết
    ↓
push mới không đến ngay
```

Triệu chứng điển hình:

> Tắt màn hình một thời gian, notification im lặng. Mở máy lên thì notification nổ một chùm.

Đây là dấu hiệu rất phù hợp với transport bị freeze / reconnect muộn.

---

## 4.4 Aurogon

Aurogon liên quan tới broadcast control và freeze policy.

Một phần quan trọng là quyền giao action:

```text
com.google.android.c2dm.intent.RECEIVE
```

đến app nhận FCM.

Điều này tạo ra một failure mode khác:

```text
FCM socket OK
GMS nhận message
    ↓
Aurogon chặn broadcast
    ↓
app đích không nhận
```

Do đó chỉ nhìn socket FCM chưa đủ.

---

## 4.5 Xiaomi Autostart AppOps

Xiaomi dùng vendor AppOps cho các quyền kiểu:

- Boot completed.
- Autostart.
- Background start activity.
- Foreground service.
- Autostart switch.

Các operation ID đã được thấy trên HyperOS hiện tại gồm:

```text
10007
10008
10021
10023
10053
```

Tuy nhiên operation ID vendor có thể đổi theo firmware.

Vì vậy app không được hardcode mù quáng mà phải probe và verify trên firmware hiện tại.

---

## 4.6 GMS network control / firewall / DNS

PowerKeeper có logic riêng dành cho Google Mobile Services.

Trên firmware được nghiên cứu, Xiaomi có thể can thiệp:

- Firewall state.
- DNS theo UID.
- Network behavior.
- Alarm.
- Wakelock.
- Backup behavior.

Vì vậy:

```text
GMS không bị freeze
```

không đồng nghĩa với:

```text
GMS vẫn có network hoàn chỉnh
```

Diagnostic engine cần tách hai thứ này.

---

## 4.7 HyperOS 4 `nightDoze`

Đây là phát hiện quan trọng cho HyperOS 4 + Android 17.

Trên một firmware HyperOS 4 China được phân tích thực tế:

- GMS đã nằm trong `MILLET_NO_RESTRICT_APP`.
- GMS limiter đã disable.
- Android Doze exemption đã có.
- Nhưng trong khoảng ban đêm, khi vào deep Doze, GMS vẫn bị freeze với reason `nightDoze`.

Framework có logic đại ý:

```text
isNight()
    +
DeviceIdleMode
    +
screen off
    ↓
AurogonImmobulusMode
    ↓
dealNightFreeze()
    ↓
freeze UID
```

Khoảng thời gian được thấy trong firmware test:

```text
23:00 → 07:59
```

Điều đặc biệt là whitelist No Restrictions không còn là exemption tuyệt đối trong đường này.

---

## 4.8 `com.miui.game.allowlist`

Trong firmware HyperOS 4 được phân tích, setting:

```text
Settings.System["com.miui.game.allowlist"]
```

được framework đọc trong lower-level Greezer pre-check.

Nếu GMS nằm trong list này, freeze request theo đường được test bị reject.

Test thực tế cho thấy:

- Không cần disable deep Doze.
- GMS giữ được trạng thái unfrozen.
- FCM socket 5228-5230 vẫn tồn tại qua các lần deep idle ngắn.

Tuy nhiên đây là **private OEM behavior**, không phải API công khai của Xiaomi.

Ngoài ra SecurityCenter / Joyose có thể sửa hoặc clear setting này trong game pre-download workflow.

Vì vậy nếu sử dụng:

```text
observe
→ preserve existing entries
→ append GMS
→ repair nếu OEM xóa
```

Không được assume một lần write là vĩnh viễn.

---

## 4.9 Xiaomi local background notification policy

Một trường hợp rất dễ bị hiểu sai:

```text
FCM đến
    ↓
app nhận data payload
    ↓
app tự build local notification
    ↓
Xiaomi chặn local notification trong background
```

Kết quả đối với người dùng vẫn là:

> "Không có notification."

Nhưng transport FCM hoàn toàn bình thường.

Diagnostic engine vì vậy phải tách:

1. Transport.
2. Delivery.
3. Execution.
4. Presentation.

---

## 5. Các key `cloud_*` được cộng đồng sử dụng

Một số script cộng đồng dùng các setting như:

```text
cloud_greezer_enable
cloud_memFreeze_control
cloud_memory_freeze_whitelist
cloud_network_priority_whitelist
cloud_lowlatency_whitelist
cloud_sleepmode_networkpolicy_enabled
```

Không nên tự động coi tất cả là fix chính thức.

Rủi ro:

- Key legacy vẫn còn tồn tại nhưng không còn consumer.
- Key được cloud service overwrite.
- Key thuộc feature khác.
- Semantic thay đổi giữa HyperOS version.
- Disable global subsystem gây hao pin hoặc side effect.

Nguyên tắc của project:

> Chỉ enable một tweak khi xác nhận firmware hiện tại có consumer thật và verify được tác dụng trước/sau.

Các key này sẽ nằm trong nhóm **experimental / probe-first**, không thuộc core fix cho tới khi có bằng chứng rõ.

---

## 6. Failure model chuẩn cho project

Project sẽ phân loại lỗi notification thành bốn tầng.

### Tầng A — Transport

Kiểm tra:

- GMS process.
- GMS frozen state.
- FCM TCP socket 5228-5230.
- Network connectivity.
- GMS firewall/DNS policy.
- Deep Doze interaction.
- Nighttime freeze.

Nếu lỗi ở đây, app đích chưa bao giờ nhận được message.

### Tầng B — Delivery

Kiểm tra:

- Aurogon broadcast rule.
- FCM receive action.
- Package stopped state.
- Broadcast restriction.
- User/profile target.

Nếu lỗi ở đây, GMS có message nhưng app đích không nhận.

### Tầng C — Execution

Kiểm tra:

- Autostart.
- Background AppOps.
- Doze policy.
- Standby bucket.
- App hibernation.
- Xiaomi battery mode.
- Process lifecycle.

Nếu lỗi ở đây, app nhận signal nhưng không có đủ quyền/thời gian xử lý.

### Tầng D — Presentation

Kiểm tra:

- POST_NOTIFICATIONS.
- Notification channels.
- Channel importance.
- App-specific notification setting.
- Xiaomi local-notification policy.
- DND / user settings.

Nếu lỗi ở đây, app xử lý xong nhưng user không nhìn thấy notification.

---

## 7. Hướng khắc phục tổng quát

Phần này chỉ mô tả mức cao. Kế hoạch triển khai chi tiết nằm trong file plan riêng.

### 7.1 Không root

Không dùng root làm requirement vì:

- App ngân hàng có thể chặn.
- Play Integrity / anti-tamper có thể bị ảnh hưởng.
- Làm tăng maintenance cost rất lớn.

### 7.2 Mô hình một APK, bootstrap đặc quyền chỉ khi thật sự bắt buộc

Ứng dụng phải được thiết kế theo nguyên tắc **APK-first**: mọi logic chẩn đoán, policy engine, guard, recovery, logging và UI đều nằm trong APK.

Nếu một số quyền hệ thống không thể được APK tự cấp do Android security model, bước bootstrap có thể là:

```text
cài APK
→ app phát hiện quyền còn thiếu
→ tạm bật Developer options + Shizuku/ADB
→ app tự thực hiện bootstrap
→ verify quyền/policy
→ tắt Shizuku
→ tắt Developer options
→ tiếp tục sử dụng chính APK
```

Shizuku không được trở thành backend bắt buộc cho thao tác hằng ngày. Nếu có tính năng chỉ chạy được khi shell identity đang tồn tại, tính năng đó phải được đánh dấu rõ là **Deep Repair / Advanced Diagnostic**, không được âm thầm làm dependency của core notification protection.

### 7.3 Persistent permission cho app

Một hướng quan trọng là cấp cho app các quyền development-level phù hợp, đặc biệt:

```text
android.permission.WRITE_SECURE_SETTINGS
```

nếu firmware cho phép.

Sau đó app có thể thao tác SettingsProvider trực tiếp mà không cần Shizuku cho các setting được phép.

Phải probe trên HyperOS 4 beta thực tế, không assume 100%.

### 7.4 Core protection

Core dự kiến gồm:

- GMS protection.
- `MILLET_NO_RESTRICT_APP` repair.
- HyperOS 4 game-allowlist protection nếu firmware xác nhận.
- Aurogon FCM delivery protection.
- GMS thaw + reconnect khi cần.
- Autostart.
- Background AppOps.
- Doze exemption có chọn lọc.
- Standby/hibernation diagnostics.
- Notification permission/channel diagnostics.

### 7.5 Không disable toàn hệ thống

Không mặc định:

- Disable Doze.
- Disable LMKD.
- Disable Cached App Freezer.
- Suspend PowerKeeper.
- Disable Greezer toàn hệ thống.

Chỉ exempt package cần thiết.

---

## 8. Nguyên tắc an toàn

Mỗi mutation phải tuân theo:

```text
read current state
→ snapshot
→ calculate minimal diff
→ apply
→ read-back verify
→ rollback nếu thất bại
```

Không dùng:

```text
settings put <key> <hardcoded full value>
```

nếu setting đang chứa state dùng chung của hệ thống.

Mọi list đều phải merge và preserve entry sẵn có.

---

## 9. Phân loại mức độ chắc chắn

### Confirmed / evidence mạnh

- Doze có thể delay normal-priority background work.
- Xiaomi PowerKeeper can thiệp background policy.
- `MILLET_NO_RESTRICT_APP` được Greezer sử dụng trên firmware đã phân tích.
- PowerKeeper có thể regenerate MILLET list.
- GMS freeze có thể làm mất FCM socket.
- Aurogon có broadcast control liên quan FCM.
- HyperOS 4 có nighttime freeze path trên firmware Android 17 đã nghiên cứu.
- `com.miui.game.allowlist` đã chặn được đường nightDoze freeze trong test.
- Android hibernation/stopped state có thể phá notification.

### Firmware-dependent

- Vendor AppOps ID.
- Greezer shell commands.
- Exact PowerKeeper schema.
- Exact Aurogon format.
- Exported thaw broadcast.
- Game allowlist behavior.
- GMS firewall implementation.
- Clone/XSpace behavior.

### Experimental

- Các `cloud_*` tweak chưa trace được consumer.
- Global freezer toggles.
- Các key cộng đồng không có evidence framework/runtime.

---

## 10. Kết luận

Vấn đề notification trên Xiaomi China ROM không phải một lỗi duy nhất.

Nó là tổ hợp của:

```text
Android power management
+
Xiaomi PowerKeeper
+
Greezer
+
Aurogon
+
GMS-specific policies
+
network policy
+
app lifecycle
+
notification presentation
```

Vì vậy một app fix đúng nghĩa phải là:

> **Một APK hoàn chỉnh chứa Notification Reliability Manager + Diagnostics Engine + Policy Repair Engine + Recovery Guard.**

Nó không phải chỉ là một nút bật Autostart, không phải wrapper cho vài lệnh ADB, và cũng không phải một bộ script yêu cầu người dùng có PC.

Thiết kế cuối cùng phải luôn probe firmware thật, apply thay đổi nhỏ nhất có thể, verify từng layer và có rollback. Công cụ ngoài APK chỉ được dùng trong quá trình phát triển hoặc bootstrap đặc quyền khi Android bắt buộc, không phải trải nghiệm sử dụng bình thường.

---

## 11. Tài liệu tham khảo chính

- Android Doze / App Standby:
  https://developer.android.com/training/monitoring-device-state/doze-standby
- Firebase message priority:
  https://firebase.google.com/docs/cloud-messaging/android-message-priority
- Android Cached App Freezer:
  https://source.android.com/docs/core/perf/cached-apps-freezer
- Android app hibernation:
  https://developer.android.com/topic/performance/app-hibernation
- HyperOS GMS/FCM/Greezer investigation:
  https://github.com/dingwen07/hyperos-fcm-fix/blob/main/docs/xiaomi-hyperos-gms-fcm-greezer-investigation.md
- HyperOS MILLET rewrite investigation:
  https://github.com/dingwen07/hyperos-fcm-fix/blob/main/docs/xiaomi-millet-no-restrict-app-rewrite-investigation.md
- HyperOS 4 nighttime FCM investigation:
  https://github.com/dingwen07/hyperos-fcm-fix/blob/main/docs/xiaomi-hyperos4-nighttime-fcm-investigation.md
