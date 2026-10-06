# HyperOS China Notification Reliability — Implementation Plan

## 1. Mục tiêu

Xây dựng **một ứng dụng Android hoàn chỉnh, phân phối dưới dạng một APK duy nhất**, dành cho Xiaomi China ROM và tập trung vào **backend notification reliability**.

### Product contract bắt buộc

Kết quả cuối cùng phải thỏa flow:

```text
User tải APK
    ↓
cài một lần
    ↓
mở app
    ↓
app tự probe thiết bị
    ↓
app hướng dẫn / thực hiện setup cần thiết
    ↓
user chọn các app cần bảo vệ
    ↓
Protection Engine chạy từ chính APK
```

Ứng dụng phải:

- Chẩn đoán chính xác notification đang hỏng ở layer nào.
- Áp dụng fix sâu nhất có thể mà không cần root.
- Chứa toàn bộ core logic trong APK: probe, diagnostics, policy engine, guard, recovery, logging và UI.
- Không yêu cầu Android Studio, Java, Gradle, ADB hay PC để sử dụng hằng ngày.
- Không biến project thành desktop script hoặc ADB wrapper.
- Nếu cần Shizuku/ADB để vượt giới hạn permission của Android, chỉ dùng chúng cho **bootstrap tạm thời / deep repair**.
- Sau bootstrap, ưu tiên để app tiếp tục vận hành khi Shizuku và Developer options đã tắt.
- Không yêu cầu app phải tự hồi sinh hoàn hảo sau reboot; user có thể mở app lại để audit/repair.
- Không phá toàn bộ cơ chế tiết kiệm pin của Android/HyperOS.
- Có rollback.
- Có log kỹ thuật đủ sâu để debug firmware mới.
- Mọi tính năng không thể hoạt động khi thiếu quyền đặc biệt phải báo rõ capability thay vì giả vờ thành công.

### Runtime dependency rule

```text
BẮT BUỘC CÀI LÂU DÀI:
- APK của project

KHÔNG ĐƯỢC LÀ DEPENDENCY HẰNG NGÀY:
- PC
- Android Studio
- ADB
- JADX
- terminal script

CHỈ ĐƯỢC DÙNG KHI THẬT SỰ CẦN BOOTSTRAP:
- Shizuku / ADB shell
```

Nếu có thể đạt cùng hiệu quả bằng persistent permission hoặc direct SettingsProvider access sau bootstrap, phải ưu tiên cách đó thay vì giữ Shizuku sống.

Target đầu tiên:

```text
Xiaomi 15 Ultra China
Android 17 / API 37
HyperOS 4 beta
```

Nhưng kiến trúc phải đủ modular để sau này hỗ trợ các build HyperOS khác.

---

# 2. Nguyên tắc thiết kế

## 2.1 Probe trước, mutate sau

Không hardcode giả định:

```text
HyperOS 3 có → HyperOS 4 chắc cũng có
```

Mỗi capability phải đi qua:

```text
detect
→ validate
→ apply
→ verify
```

Nếu không verify được:

```text
Unsupported / Experimental
```

không được báo "Fixed".

---

## 2.2 Minimal mutation

Không disable global subsystem nếu chỉ cần exempt 1 package.

Ưu tiên:

```text
per-package exemption
>
per-package AppOp
>
shared allowlist merge
>
global toggle
```

Global toggle chỉ dùng khi có bằng chứng rõ và user chủ động bật experimental mode.

---

## 2.3 Read-modify-write

Mọi shared setting phải:

```text
read
→ parse
→ preserve existing entries
→ merge desired entries
→ write
→ verify
```

Không overwrite list dùng chung.

---

## 2.4 Idempotent

Chạy Fix nhiều lần phải cho cùng kết quả.

Ví dụ:

```text
applyProtection()
applyProtection()
applyProtection()
```

không được tạo duplicate, phá format hoặc thay đổi state không liên quan.

---

## 2.5 Có rollback

Mỗi mutation cần lưu:

- Original value.
- App-added value.
- Firmware/build fingerprint.
- Timestamp.
- Verification result.

Khi disable protection:

- Chỉ remove phần app đã thêm.
- Không xóa state vốn tồn tại trước.

---

# 3. Kiến trúc tổng thể

```text
                         UI
                          │
                          ↓
                  Protection Controller
                          │
       ┌──────────────────┼──────────────────┐
       ↓                  ↓                  ↓
    RomProbe       PersistentGuard     DeepRepairEngine
       │                  │                  │
       ↓                  ↓                  ↓
 capability map     Settings/API        Shizuku shell
       │                  │                  │
       └──────────────────┼──────────────────┘
                          ↓
                    Policy Engine
                          │
            ┌─────────────┼─────────────┐
            ↓             ↓             ↓
          GMS         Xiaomi OEM      Target apps
            │             │             │
       FCM health    Greezer/Aurogon   background
       sockets       PowerKeeper       autostart
       reconnect     MILLET            hibernation
                    nightDoze          notification
```

Modules:

```text
app/
core/
  probe/
  diagnostics/
  policy/
  persistence/
  logging/
privileged/
  shizuku/
  shell/
xiaomi/
  powerkeeper/
  greezer/
  aurogon/
  millet/
  nightdoze/
android/
  doze/
  appops/
  standby/
  hibernation/
  notifications/
```

Không nhất thiết phải tạo đúng package structure này ngay từ commit đầu, nhưng boundary logic phải tương tự.

---

# 4. Phase 0 — Project foundation

## 4.0 Build và distribution pipeline

Codex phải dựng project để **repo tự build được APK**, không phụ thuộc máy người dùng đã cài Android Studio.

Bắt buộc có:

- Gradle Wrapper commit trong repo.
- GitHub Actions workflow build debug APK ở mỗi pull request/push phù hợp.
- Workflow build release artifact khi tạo tag/release.
- Artifact APK có tên/version rõ ràng.
- Build từ clean checkout phải chạy được trên CI.
- Không commit local SDK path, signing secret hoặc file máy cá nhân.

Development pipeline mong muốn:

```text
Codex sửa code
→ push GitHub
→ GitHub Actions build
→ APK artifact
→ cài APK lên Xiaomi thật để test
```

Android Studio chỉ là công cụ tùy chọn cho developer, không phải requirement để build/test flow cơ bản.

## 4.1 Tạo Android project

Khuyến nghị:

- Kotlin.
- Gradle Kotlin DSL.
- minSdk đủ thấp để test trên nhiều HyperOS nếu cần.
- compileSdk / targetSdk theo Android 17 SDK khi toolchain hỗ trợ chính thức.
- Jetpack Compose hoặc XML đều được, UI không phải trọng tâm.

Không hạ targetSdk xuống 22 làm nền tảng chính.

Nếu sau này cần legacy behavior để test một edge case, đặt thành flavor riêng, không biến production app thành legacy-target app.

---

## 4.2 Logging

Phải có structured log ngay từ đầu.

Mỗi event:

```text
timestamp
component
operation
target package
before
after
result
error
firmware fingerprint
```

Ví dụ:

```text
09:21:12 MILLET Repair com.google.android.gms absent→present OK
09:21:13 GMS FCM socket 0→1 RECOVERED
```

Cho phép export log dạng text/json.

Không log dữ liệu notification, token FCM, account hoặc nội dung riêng tư.

---

# 5. Phase 1 — ROM Probe

Đây là phase quan trọng nhất trước khi fix.

## 5.1 Device fingerprint

Thu thập:

- Manufacturer.
- Model.
- Device codename.
- Android version.
- SDK.
- HyperOS version.
- Build fingerprint.
- Security patch.
- PowerKeeper version.
- SecurityCenter version.
- Joyose version.
- GMS version.

Không upload tự động.

---

## 5.2 Package probe

Kiểm tra tồn tại:

```text
com.miui.powerkeeper
com.miui.securitycenter
com.xiaomi.joyose
com.google.android.gms
com.android.vending
```

Và package/component tương ứng nếu firmware đổi tên.

---

## 5.3 Settings capability probe

Read-only probe:

```text
MILLET_NO_RESTRICT_APP
aurogon_enable
com.miui.game.allowlist
các cloud_* candidate
```

Phân loại:

- key exists.
- key absent.
- readable.
- writable with app permission.
- writable only shell.
- observer supported.

Không tạo key experimental chỉ để test nếu chưa cần.

---

## 5.4 Greezer probe

Khi Shizuku có mặt:

- `dumpsys greezer`
- available command help nếu có.
- freeze history.
- GMS frozen state.
- supported commands.

Không assume command từ firmware khác hoạt động.

---

## 5.5 AppOps probe

Query toàn bộ relevant ops:

- Standard background AppOps.
- Xiaomi vendor ops.

Xác nhận ID tương ứng với:

- autostart.
- background start.
- boot completed.
- foreground service.

Nếu ID không match behavior expected → mark unsupported.

---

# 6. Phase 2 — Bootstrap permission model

Mục tiêu là **một APK hoạt động độc lập tối đa có thể**. Shizuku/ADB chỉ là bootstrap privilege source khi Android không cho APK tự nâng quyền. Core architecture không được xây quanh việc Shizuku luôn tồn tại.

## 6.1 Capability levels

App có 3 mode:

### Level 0 — Normal APK

Không Shizuku.

Có thể:

- diagnostics public APIs.
- notification permission/channel checks.
- ContentObserver với setting có quyền đọc.
- mutate những setting app được phép.

### Level 1 — Persistent privileged grants

Đã bootstrap.

Candidate:

```text
WRITE_SECURE_SETTINGS
```

và các permission development-level khác nếu thật sự cần và grant được.

Sau grant:

- verify bằng PackageManager.
- reboot test.
- Developer options OFF test.
- Shizuku OFF test.

Nếu permission biến mất hoặc API vẫn bị vendor chặn → không coi là supported.

### Level 2 — Live shell

Shizuku đang chạy.

Dùng cho:

- dumpsys.
- shell AppOps.
- deviceidle command.
- Greezer runtime state.
- deep diagnostics.
- repair state shell-only.

---

## 6.2 Bootstrap flow

Bootstrap phải được điều khiển từ chính APK. User không được yêu cầu copy/paste hàng loạt lệnh shell như flow sản phẩm chính.

```text
Install APK
    ↓
Open app
    ↓
ROM Probe
    ↓
Core có đủ quyền?
   ↙          ↘
 YES          NO
  ↓            ↓
run        app hiển thị Bootstrap Wizard
             ↓
       temporary Shizuku/ADB privilege
             ↓
       grant persistent permissions
             ↓
       apply shell-only baseline
             ↓
       verify every mutation
             ↓
       store capability matrix
             ↓
       Shizuku có thể tắt
       Developer options có thể tắt
             ↓
       APK tiếp tục vận hành
```

Không được ép user giữ Developer options bật. Nếu bootstrap không khả dụng, app vẫn phải chạy diagnostics và các protection mà quyền hiện tại cho phép, đồng thời hiển thị rõ phần nào bị giới hạn.

---

## 6.3 Permission persistence verification

Đây là checkpoint bắt buộc.

Test:

1. Grant.
2. Kill app.
3. Open lại.
4. Stop Shizuku.
5. Disable Developer options.
6. Open lại.
7. Reboot.
8. Open lại.

Ghi kết quả theo permission.

Không suy luận "pm grant thành công = vĩnh viễn".

---

# 7. Phase 3 — Diagnostics engine

Tạo một health model thống nhất.

```text
HEALTHY
DEGRADED
BROKEN
UNKNOWN
UNSUPPORTED
```

## 7.1 Transport health

Check:

- GMS installed/enabled.
- GMS process.
- frozen state nếu accessible.
- FCM socket state nếu accessible.
- network.
- game allowlist protection.
- MILLET protection.
- nighttime risk.

## 7.2 Delivery health

Check:

- Aurogon.
- FCM receive rules.
- package stopped state nếu accessible.
- user/profile mapping.

## 7.3 Execution health

Check:

- Autostart.
- background AppOps.
- Doze whitelist.
- standby bucket.
- battery optimization.
- hibernation.

## 7.4 Presentation health

Check:

- POST_NOTIFICATIONS.
- channel count.
- blocked channels.
- importance.
- app-wide notification enable.
- Xiaomi-specific presentation policy nếu probe được.

---

# 8. Phase 4 — GMS core protection

Đây là ưu tiên P0.

## 8.1 MILLET controller

Implement parser cho:

```text
MILLET_NO_RESTRICT_APP
```

Requirements:

- preserve whitespace/format nếu cần.
- parse package list đúng format firmware.
- append GMS.
- không duplicate.
- atomic-like serialized mutations trong app.
- read-back verify.

State cần lưu:

```text
wasPresentBeforeActivation
appAddedGms
lastObservedValue
lastRepairTime
```

---

## 8.2 MILLET observer

Ưu tiên event-driven:

```text
ContentObserver
```

Nếu observer hoạt động ổn trong app process:

```text
OEM rewrite
→ callback
→ debounce
→ read
→ repair
```

Nếu firmware không notify đáng tin cậy:

fallback:

- periodic lightweight check.
- WorkManager safety net.
- optional foreground guard mode.

Không poll 2.5s mặc định nếu không cần.

---

## 8.3 Repair latency test

PowerKeeper có thể rewrite list và Greezer freeze sau đó.

Test thực tế:

- trigger battery-policy change.
- đo từ lúc setting mất GMS tới freeze attempt.
- đo observer latency.
- xác định polling fallback nếu cần.

Target:

```text
repair before shortest observed freeze window
```

---

# 9. Phase 5 — HyperOS 4 nightDoze protection

Ưu tiên P0 cho HyperOS 4.

## 9.1 Probe game allowlist

Read:

```text
Settings.System["com.miui.game.allowlist"]
```

Xác nhận parser của firmware hiện tại.

Firmware nghiên cứu dùng underscore:

```text
pkg1_pkg2_pkg3
```

Không assume Xiaomi 15 Ultra beta giống hệt.

---

## 9.2 Safe merge

Nếu confirmed:

```text
existing list
+ com.google.android.gms
```

Preserve toàn bộ entry khác.

Store whether GMS existed trước activation.

---

## 9.3 Framework reload

Không assume stored value = runtime cache updated.

Phải test:

- write changed value.
- check Greezer behavior.
- check freeze history.
- check socket.

Nếu same-value write không trigger observer, cần minimal semantic-preserving toggle phù hợp với parser firmware.

Chỉ implement sau khi test thực tế.

---

## 9.4 Thaw + reconnect

Sau khi repair exemption:

1. Thaw GMS nếu firmware có exported Xiaomi helper và probe xác nhận.
2. Request GCM reconnect.
3. Verify process.
4. Verify FCM socket.

Không coi broadcast result code là bằng chứng thành công.

---

# 10. Phase 6 — Aurogon delivery protection

## 10.1 Parse shared setting

Không replace toàn setting.

Cần parser/serializer riêng.

Preserve:

- OEM sections.
- package actions.
- flags khác.

App thêm một managed section có marker riêng.

Ví dụ concept:

```text
our.package/__MANAGED_MARKER__
```

để rollback chính xác.

---

## 10.2 Target packages

Chỉ áp dụng cho app user chọn.

Đảm bảo action:

```text
com.google.android.c2dm.intent.RECEIVE
```

được phép nếu firmware đúng cơ chế này.

---

## 10.3 Verify delivery

Không chỉ kiểm tra setting.

Nếu có thể:

- logs broadcast.
- app process wake.
- notification test endpoint.

Sau này có thể thêm companion test app để gửi FCM synthetic test.

---

# 11. Phase 7 — Per-app policy manager

Mỗi app có policy object.

Ví dụ:

```text
AppPolicy
  packageName
  protectFcmDelivery
  protectAutostart
  backgroundMode
  dozeMode
  hibernationProtection
  notificationDiagnostics
```

Không bắt tất cả app dùng cùng preset.

---

## 11.1 Preset

### Balanced

- Autostart.
- Aurogon.
- background AppOps.
- no aggressive Doze exemption.

### Reliable

- Balanced.
- Doze whitelist.
- standby active repair.
- hibernation protection.

### Maximum

- Reliable.
- extra Xiaomi-specific verified policies.
- more aggressive guard.

Banking app mặc định dùng Balanced trước, vì không phải app nào cũng cần được giữ sống liên tục.

---

# 12. Phase 8 — Autostart / background AppOps

## 12.1 Standard AppOps

Candidate:

```text
RUN_IN_BACKGROUND
RUN_ANY_IN_BACKGROUND
```

Apply only if current mode restricts selected app.

Verify after write.

---

## 12.2 Xiaomi AppOps

Probe vendor ops.

Không hardcode 10008/10053 nếu current build trả semantic khác.

Cần:

```text
discover
→ test package
→ change
→ verify UI/behavior
```

Lưu mapping theo firmware fingerprint.

---

# 13. Phase 9 — Doze / standby / hibernation

## 13.1 DeviceIdle whitelist

Chỉ cho app critical.

Không whitelist toàn bộ app installed.

## 13.2 Standby bucket

Có thể set ACTIVE trong deep repair.

Nhưng system có quyền tự thay đổi lại.

Vì vậy:

- diagnostics first.
- repair when severely degraded.
- không spam write.

## 13.3 Hibernation

Detect:

- unused app state.
- auto revoke state.
- package hibernated.

Nếu có public/privileged API hợp lệ thì disable cho protected app.

Nếu chỉ shell có thể sửa, đưa vào Deep Repair.

---

# 14. Phase 10 — GMS network diagnostics

Không modify network bừa bãi.

Check:

- connectivity.
- DNS behavior.
- UID policy.
- Data Saver.
- background data.
- PowerKeeper GMS state nếu dump accessible.
- FCM socket.

Nếu phát hiện OEM firewall:

- tìm actual control path trên firmware.
- trace writer.
- chỉ sau đó mới implement fix.

---

# 15. Phase 11 — Notification presentation diagnostics

Per app:

- notification permission.
- app-wide enable.
- channels.
- channel importance.
- blocked channel.
- bubbles không phải priority.
- DND status.

Xiaomi-specific local notification policy:

- probe AppOps/service.
- read-only trước.
- chỉ apply sau khi xác nhận semantic.

---

# 16. Phase 12 — Experimental cloud_* lab

Tách hoàn toàn khỏi stable engine.

Candidate:

```text
cloud_greezer_enable
cloud_memFreeze_control
cloud_memory_freeze_whitelist
cloud_network_priority_whitelist
cloud_lowlatency_whitelist
cloud_sleepmode_networkpolicy_enabled
```

Muốn promote một key sang stable phải có đủ:

1. Consumer được tìm thấy trong framework/APK.
2. Runtime read được chứng minh.
3. Before/after behavior đo được.
4. Không gây side effect nghiêm trọng.
5. Rollback hoạt động.

Không đáp ứng đủ → giữ Experimental.

---

# 17. Phase 13 — Reboot behavior

Không bắt buộc v1 phải tự chạy hoàn hảo ngay sau boot. Tuy nhiên **reboot không được biến app thành tool cần PC để phục hồi**.

Flow chấp nhận được:

```text
reboot
    ↓
user mở chính APK
    ↓
Quick Audit
    ↓
persistent state còn?
   ↙          ↘
 YES          NO
  ↓            ↓
continue    app tự repair bằng quyền đã có
               ↓
          còn shell-only state bị reset?
               ↓
          chỉ lúc đó mới đề nghị
          temporary Deep Repair
```

Health screen phải nói rõ:

```text
Persistent protection: OK
Runtime Greezer flag: reset
Core protection usable without Shizuku: YES/NO
Deep Repair currently required: YES/NO
```

Mục tiêu là sau reboot đa số chức năng vẫn chạy chỉ bằng APK. Deep Repair không được là việc user phải làm sau mọi lần khởi động nếu có cách persistent hơn.

---

# 18. Phase 14 — Self-protection cho chính app

App guard cũng có thể bị Xiaomi kill.

Apply cho chính package:

- Autostart.
- background AppOps.
- battery unrestricted nếu cần.
- Doze whitelist nếu guard realtime được bật.

Nhưng design vẫn phải chịu được việc process chết.

State phải nằm trong persistent storage.

Khi process mở lại:

```text
load desired policy
→ audit
→ reconcile
```

---

# 19. Phase 15 — Scheduler strategy

Ưu tiên event-driven.

Thứ tự:

1. ContentObserver.
2. Broadcast/system event.
3. App lifecycle event.
4. WorkManager safety net.
5. Polling chỉ khi không còn signal tin cậy.

Không dùng:

- permanent wake lock.
- alarm mỗi vài giây.
- infinite foreground service mặc định.

Nếu cần realtime guard mode:

- user bật chủ động.
- hiển thị battery tradeoff.
- đo CPU trước khi release.

---

# 20. Phase 16 — Test suite

## 20.1 Unit tests

Parser:

- MILLET.
- Aurogon.
- game allowlist.
- rollback merge.

Test:

- empty.
- null.
- malformed.
- duplicate.
- unknown OEM entry.
- concurrent-like mutation.

---

## 20.2 Instrumented tests

- permission state.
- SettingsProvider read/write.
- ContentObserver.
- PackageManager.
- NotificationManager.

---

## 20.3 Device tests

Trên Xiaomi 15 Ultra:

### Test A — baseline

- Không fix.
- screen off.
- 30m / 1h / 3h.
- ghi freeze/socket.

### Test B — MILLET only

- apply MILLET.
- repeat.

### Test C — MILLET + nightDoze exemption

- repeat deep idle.

### Test D — Aurogon

- verify delivery.

### Test E — full reliable preset

- test Messenger/Instagram/Gmail.

---

## 20.4 Night test

Bắt buộc test:

```text
23:00 → 08:00
```

vì HyperOS 4 có nighttime policy.

Không chỉ test bằng force-idle ban ngày.

---

## 20.5 Real notification tests

Các app:

- Messenger.
- Instagram.
- Gmail.
- 1-2 banking app.
- app dùng local notification từ data payload nếu xác định được.

Test matrix:

```text
screen on
screen off 5m
screen off 30m
deep idle
Wi-Fi
mobile data
battery saver
nighttime
after reboot
after changing battery policy
```

---

# 21. Phase 17 — Battery impact benchmark

Không release "Maximum protection" nếu chưa đo.

Đo:

- GMS CPU.
- guard CPU.
- wakelock.
- overnight battery.
- reconnect count.
- settings write count.

Goal:

- Healthy state gần như read-only.
- Không write lặp.
- Không wake device vô ích.

---

# 22. Phase 18 — Failure and recovery

Mỗi module phải có state machine.

Ví dụ MILLET:

```text
UNKNOWN
  ↓
READ
  ↓
HEALTHY
  └─ OEM rewrite → DEGRADED
                      ↓
                   REPAIR
                    ↓  ↓
                  OK   FAILED
```

Nếu failed:

- log.
- retry bounded.
- không tight-loop.
- không repeatedly overwrite.

---

# 23. Phase 19 — Rollback

Tạo màn hình:

```text
Restore original state
```

Rollback order:

1. Stop guard.
2. Remove app-managed Aurogon section.
3. Remove GMS from shared list chỉ khi app là bên đã add.
4. Restore AppOps changed by app.
5. Restore Doze state.
6. Clear stored snapshots.

Không revoke `WRITE_SECURE_SETTINGS` tự động nếu việc revoke cần shell và user chưa yêu cầu.

Có thể cung cấp Deep Cleanup khi Shizuku đang available.

---

# 24. Phase 20 — Minimal UI

UI chỉ cần:

## Home

```text
Notification Health      92%

GMS Transport            Healthy
Xiaomi Freezer           Protected
FCM Delivery             Healthy
Protected Apps           6

[Quick Repair]
[Deep Repair]
[Apps]
[Diagnostics]
```

## App detail

```text
Messenger

Autostart                 OK
Background                OK
Doze                      OK
Aurogon FCM               OK
Hibernation               OK
Notification channels     OK
```

Không cần animation cầu kỳ.

Backend quan trọng hơn.

---

# 25. Phase 21 — Release gates

Không release stable nếu chưa đạt:

- No destructive global tweak by default.
- Rollback passes.
- Xiaomi shared lists preserved.
- No tight loop.
- No permanent wake lock.
- No root dependency.
- Core works after Shizuku disabled where expected.
- Permission persistence verified.
- Nighttime HyperOS 4 test completed.
- Real push delivery improved compared to baseline.

---

# 26. Double-check kiến trúc

Sau khi rà lại toàn bộ plan, các kết luận cần giữ như sau.

## 26.1 Điều có thể kỳ vọng hoạt động không Shizuku sau bootstrap

Khả năng cao:

- App-owned ContentObserver.
- Direct SettingsProvider access nếu persistent grant thực sự usable.
- MILLET repair.
- game allowlist repair nếu setting writable.
- Aurogon repair nếu setting writable.
- Notification diagnostics.
- Normal Android APIs.

Nhưng phải verify trên chính HyperOS 4 beta.

---

## 26.2 Điều không nên giả định hoạt động sau Shizuku tắt

- `dumpsys greezer`.
- shell-only freeze state.
- Greezer runtime command.
- arbitrary `appops set`.
- `cmd deviceidle`.
- clear stopped state bằng shell.
- protected system-service calls.

Các tính năng này thuộc Deep Repair hoặc diagnostics nâng cao.

---

## 26.3 Điều không nên coi là persistent qua reboot nếu chưa test

- Greezer runtime limiter.
- LM allowlist runtime state.
- standby bucket.
- vendor cache.
- framework in-memory whitelist cache.

Reboot audit bắt buộc.

---

## 26.4 Điều có thể persist nhưng vẫn bị OEM overwrite

- MILLET.
- game allowlist.
- Aurogon.
- một số Xiaomi settings.

Do đó persistence trên disk không có nghĩa là ổn định vĩnh viễn.

Guard cần reconciliation.

---

## 26.5 Không phụ thuộc một single magic fix

Không có:

```text
MILLET = solved
```

hay:

```text
Autostart = solved
```

Core fix phải bao phủ ít nhất:

```text
GMS transport
+
Xiaomi freeze exemption
+
FCM delivery
+
target app background policy
+
notification presentation
```

---

# 27. Thứ tự implementation đề xuất

Thứ tự code thực tế:

1. Project skeleton + Gradle Wrapper.
2. GitHub Actions build APK.
3. Logging.
4. ROM Probe.
5. Capability model.
6. In-app bootstrap wizard.
7. Shizuku integration chỉ cho temporary privilege.
8. Persistent permission verification.
9. Diagnostics model.
10. MILLET controller.
11. HyperOS 4 game allowlist controller.
12. GMS thaw/reconnect.
13. Aurogon controller.
14. Per-app background policy.
15. Doze/standby/hibernation.
16. Notification diagnostics.
17. Network diagnostics.
18. Standalone-without-Shizuku validation.
19. Reboot audit.
20. Rollback.
21. Experimental cloud_* lab.
22. Battery benchmark.
23. Real overnight testing.
24. Release APK + stable release.

Không đảo thứ tự bằng cách làm UI trước.

---

# 28. Definition of Done cho v1

v1 được coi là đạt khi:

- Repo clean checkout tự build được APK bằng Gradle/GitHub Actions.
- User chỉ cần cài **APK của project** làm ứng dụng lâu dài.
- Toàn bộ thao tác chính được thực hiện từ UI của APK, không yêu cầu chạy script tay trên PC.
- Lần mở đầu app tự probe ROM và tạo capability matrix.
- Nếu cần bootstrap đặc quyền, app hướng dẫn và điều khiển flow đó; Shizuku/ADB chỉ là phương tiện tạm thời.
- Sau bootstrap có thể tắt Shizuku.
- Sau bootstrap có thể tắt Developer options.
- Core protection vẫn hoạt động ở mức đã xác nhận khi chỉ còn APK chạy.
- App vẫn audit và repair được các persistent core policy đã xác nhận.
- Sau reboot, mở lại APK là đủ cho Quick Audit/normal repair; PC không phải flow recovery mặc định.
- Messenger/Gmail/Instagram nhận push ổn định hơn baseline khi screen off.
- GMS không bị HyperOS 4 nighttime freezer giết trong test đã support.
- Không root.
- Không phá app ngân hàng.
- Không disable Doze toàn máy.
- Không disable LMKD.
- Không disable toàn bộ freezer.
- Có rollback.
- Có diagnostics đủ để biết failure ở layer nào.
- Mọi tính năng còn cần live Shizuku phải được tách thành Deep Repair/Advanced Diagnostic và không được giả là standalone.

---

# 29. Những việc để sau v1

- XSpace / clone-profile recovery hoàn chỉnh.
- Work profile.
- Global ROM support.
- Samsung/Oppo/Vivo.
- Synthetic FCM test server.
- Desktop ADB companion.
- Automatic firmware signature database.
- Community capability profiles.

---

# 30. Tài liệu kỹ thuật dùng làm baseline

- Android Doze / App Standby:
  https://developer.android.com/training/monitoring-device-state/doze-standby
- Firebase message priority:
  https://firebase.google.com/docs/cloud-messaging/android-message-priority
- Android app hibernation:
  https://developer.android.com/topic/performance/app-hibernation
- HyperOS GMS/FCM/Greezer:
  https://github.com/dingwen07/hyperos-fcm-fix/blob/main/docs/xiaomi-hyperos-gms-fcm-greezer-investigation.md
- MILLET rewrite:
  https://github.com/dingwen07/hyperos-fcm-fix/blob/main/docs/xiaomi-millet-no-restrict-app-rewrite-investigation.md
- HyperOS 4 nightDoze:
  https://github.com/dingwen07/hyperos-fcm-fix/blob/main/docs/xiaomi-hyperos4-nighttime-fcm-investigation.md
- Upstream implementation:
  https://github.com/dingwen07/hyperos-fcm-fix

---

# 31. Chỉ thị triển khai cho Codex

Khi Codex bắt tay vào code, ưu tiên **tạo sản phẩm chạy được**, không dừng ở research hoặc sinh script.

Mỗi phase phải để repo ở trạng thái buildable. Khi gặp API private/firmware-dependent:

1. Implement interface/capability boundary trước.
2. Probe runtime.
3. Chỉ enable mutation nếu probe xác nhận.
4. Có fallback `UNSUPPORTED` thay vì crash.
5. Không chặn việc build APK chỉ vì một tweak vendor chưa xác nhận.

Không được chuyển kiến trúc thành:

```text
Android app UI
    ↓
gọi một đống script yêu cầu PC
```

Kiến trúc đúng phải là:

```text
APK
├── ROM Probe
├── Diagnostics
├── Persistent Guard
├── Policy Engine
├── Recovery Engine
├── Shizuku Bootstrap Adapter (temporary)
└── UI
```

Mục tiêu sau cùng là **release APK có thể cài trực tiếp lên Xiaomi và sử dụng như một app**, không phải bộ công cụ dành cho developer.
