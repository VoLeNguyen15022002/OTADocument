# Flash Bootloader (FBL) — Tài Liệu Tổng Hợp Kỹ Thuật

Tài liệu tổng hợp kiến thức cốt lõi và thực chiến về **Flash Bootloader (FBL)** trên vi điều khiển nhúng (ARM Cortex-M & Automotive MCU như Infineon AURIX TC397), bao gồm: ràng buộc phần cứng Flash, bố cục bộ nhớ & chuỗi khởi động, quy trình nhảy sang Application, giao thức nạp UDS/CanTp, an toàn mất điện & rollback, và Secure Boot / HSM.

---

## Mục Lục

1. [Ràng buộc phần cứng bộ nhớ Flash](#1-ràng-buộc-phần-cứng-bộ-nhớ-flash)
2. [Memory Layout & Chuỗi khởi động (Boot Sequence)](#2-memory-layout--chuỗi-khởi-động-boot-sequence)
3. [Quy trình nhảy từ FBL sang Application (Jump-to-App)](#3-quy-trình-nhảy-từ-fbl-sang-application-jump-to-app)
4. [Giao thức nạp Firmware (UDS ISO 14229 & CanTp ISO 15765-2)](#4-giao-thức-nạp-firmware-uds-iso-14229--cantp-iso-15765-2)
5. [An toàn mất điện, toàn vẹn dữ liệu & Rollback](#5-an-toàn-mất-điện-toàn-vẹn-dữ-liệu--rollback)
6. [Secure Boot, Chữ ký số & HSM](#6-secure-boot-chữ-ký-số--hsm)
7. [Bảng tra cứu nhanh các lỗi thực tế hay gặp](#7-bảng-tra-cứu-nhanh-các-lỗi-thực-tế-hay-gặp)

---

## 1. Ràng buộc phần cứng bộ nhớ Flash

### 1.1 Erase-before-Write & Độ mịn Xóa/Ghi
- Khác với RAM, bộ nhớ Flash **không cho phép ghi đè trực tiếp** từng byte bất kỳ — bắt buộc phải **xóa (erase)** về trạng thái mặc định (`0xFF`) trước khi ghi mới.
- **Đơn vị xóa tối thiểu (Erase Sector)** lớn hơn nhiều so với **đơn vị ghi tối thiểu (Write Page / Phrase)**:
  - Ví dụ: `Erase Sector = 16 KB`, `Write Page = 32 byte` (có kèm ECC).
  - Muốn nạp một App nặng **50 KB**, bắt buộc phải xóa tối thiểu **`4 sector = 64 KB`** (vì `3 sector = 48 KB < 50 KB`).
- **Padding & ECC:**
  - Nếu mảnh dữ liệu cuối cùng lẻ (ví dụ chỉ còn `10 byte` trong page `32 byte`), FBL phải **đệm (pad)** `22 byte` còn thiếu trong buffer (thường bằng `0xFF`) để ghi đủ nguyên page `32 byte` nhằm tính **ECC** hợp lệ.
  - Trên Flash có ECC (như PFLASH của TC397), ghi đè 2 lần vào cùng một page chưa xóa sẽ gây **lỗi ECC** ngay lập tức.

### 1.2 Same-Bank Read-While-Write (RWW) & Flash Driver trên RAM
- Một bank Flash vật lý **không thể đồng thời** đặt điện áp cao để xóa/ghi và phục vụ **instruction fetch** (CPU đọc lệnh máy) trên **cùng bank đó**:
  - Trên MCU **multi-bank** (ví dụ AURIX TC397 có `PF0`–`PF5`), CPU chạy ở `PF0` có thể xóa/ghi `PF1` (khác bank).
  - Trên MCU **single-bank** (hoặc khi xóa/ghi **chính bank đang chạy lệnh**), chạy hàm xóa trực tiếp từ Flash sẽ làm đứng CPU hoặc **BusFault**.
- **Giải pháp chuẩn (HIS Flash Driver):** Nạp và thực thi **Flash Driver từ RAM** (ví dụ `SRAM` trên Cortex-M, `PSPR` trên TC397).
  - **Lợi ích 1 (Phần cứng):** CPU fetch lệnh xóa/ghi từ RAM, hoàn toàn không xung đột với bank Flash đang bận.
  - **Lợi ích 2 (An toàn):** Trong vận hành bình thường ngoài đường, Flash Driver **không tồn tại trong ECU** (chỉ được tải vào RAM khi vào Programming Session và mất sạch sau reset), ngăn ngừa code lỗi (runaway code) vô tình nhảy vào hàm xóa Flash.

### 1.3 Read-Modify-Write & Rủi ro mất điện
- Để sửa chỉ `1 byte` trong sector `16 KB`, kỹ thuật **Read-Modify-Write** đọc toàn bộ `16 KB` ra RAM → sửa `1 byte` → xóa sector `16 KB` → ghi lại từ RAM.
- **Nhược điểm chí mạng:** Nếu mất điện nguồn chính (rút ắc quy — cả Retention RAM cũng mất) ngay sau khi xóa sector và trước khi ghi lại, toàn bộ `16 KB` sẽ mất trắng.
- **Khắc phục cho dữ liệu cấu hình/metadata quan trọng:** Dùng **2 sector Flash (Ping-Pong / A-B)** kết hợp `version/sequence counter` + `CRC` + `commit flag` ghi cuối cùng.

---

## 2. Memory Layout & Chuỗi khởi động (Boot Sequence)

### 2.1 Cơ chế phần cứng khi Reset (ARM Cortex-M)
Ngay khi thoát Power-On Reset (POR) hoặc Soft Reset, **phần cứng CPU** tự động thực hiện trước khi có bất kỳ dòng code C nào chạy:

| Địa chỉ đầu Flash | Ví dụ giá trị | Hành động phần cứng |
|---|---|---|
| `0x08000000` (`+0x00`) | `0x20005000` | Nạp vào **Main Stack Pointer (`MSP`)** — đỉnh Stack trong SRAM |
| `0x08000004` (`+0x04`) | `0x08000199` | Nạp vào **Program Counter (`PC`)** — địa chỉ `Reset_Handler` (bit 0 = `1` báo chế độ Thumb) |

Trình tự khởi động đầy đủ:
```text
POR / Reset
  └─► Hardware đọc MSP tại 0x08000000
  └─► Hardware đọc Reset Vector tại 0x08000004 → nạp PC
  └─► Reset_Handler chạy: copy .data (Flash → RAM), xóa .bss = 0, gọi SystemInit() (clock)
  └─► main() của FBL bắt đầu chạy
```

> **Tại sao mất Sector 0 làm ECU bị brick?**  
> Flash vừa xóa chứa toàn `0xFF`. CPU đọc `MSP = 0xFFFFFFFF` và `PC = 0xFFFFFFFF` → lập tức fault; nhưng HardFault vector cũng nằm trong sector 0 (`0xFFFFFFFF`) → CPU rơi vào **Lockup**. Lúc này chỉ cứu được bằng **BootROM** (ví dụ kéo chân `BOOT0`) hoặc nạp lại qua **SWD/JTAG/DAP**.

### 2.2 Phân vùng bộ nhớ & Linker Script
```text
FLASH (256 KB: 0x08000000 – 0x0803FFFF)
├── 0x08000000 – 0x08007FFF  (32 KB)   [ FBL — có Vector Table riêng ]
└── 0x08008000 – 0x0803FFFF  (224 KB)  [ Application — có Vector Table riêng ]

RAM (64 KB: 0x20000000 – 0x2000FFFF)
├── 0x20000000 – 0x2000FFEF  (0xFFF0)  [ RAM chính — FBL và App dùng toàn bộ khi đến lượt ]
└── 0x2000FFF0 – 0x2000FFFF  (16 byte) [ Shared noinit RAM — truyền cờ Programming Session ]
```

Ví dụ cấu hình `MEMORY` trong Linker Script của **Application**:
```ld
MEMORY
{
  FLASH       (rx)  : ORIGIN = 0x08008000, LENGTH = 224K
  RAM         (xrw) : ORIGIN = 0x20000000, LENGTH = 0xFFF0
  SHARED_RAM  (rw)  : ORIGIN = 0x2000FFF0, LENGTH = 16
}
```

**Những nguyên tắc sống còn về Memory Layout:**
1. **Phải rebuild App với Linker Script mới:** Nếu nạp file `.bin` cũ (build cho `0x08000000`) vào `0x08008000`, mọi địa chỉ tuyệt đối (`Reset_Handler`, ISR, con trỏ hằng) đều trỏ ngược về vùng `0x08000xxx` của FBL → crash ngay.
2. **FBL KHÔNG "sleep" trong RAM:** FBL và App không bao giờ chạy song song trên MCU bare-metal. Khi App chạy, App được quyền dùng gần như **toàn bộ RAM**; khi cần quay về FBL (nhận lệnh `10 02 Programming Session`), App ghi magic flag vào vùng **`noinit` RAM** (vài chục byte, đặt cùng địa chỉ cố định ở cả 2 linker script) rồi thực hiện **Soft Reset**. FBL đọc cờ xong phải **xóa ngay** để tránh kẹt ở FBL hoặc bị rác RAM kích hoạt nhầm khi POR.
3. **Bảo vệ Sector 0 phải bằng phần cứng:** Linker script chỉ có tác dụng lúc **build**, không chặn được code lỗi ghi trực tiếp thanh ghi `FLASH->CR` lúc **runtime**. Phải dùng **Hardware Write Protection (WRP)** kết hợp **khóa vĩnh viễn** (RDP Level 2 / OTP fuse) hoặc tách quyền bằng **HSM / TrustZone** (lưu ý: `OPTLOCK` mặc định sau reset chỉ là khóa tạm, code privileged vẫn mở lại được bằng `FLASH->OPTKEYR`).

---

## 3. Quy trình nhảy từ FBL sang Application (Jump-to-App)

Trước khi nhảy, FBL **bắt buộc** xác nhận App hợp lệ (Valid flag / CRC / CMAC, và kiểm tra `app_sp` nằm trong dải SRAM hợp lệ). Sau đó thực hiện **đúng thứ tự** 5 bước:

1. **Tắt ngắt toàn cục (`__disable_irq()`) đầu tiên:** Chặn tuyệt đối mọi ngắt xen ngang trong lúc chuyển giao.
2. **Dừng `SysTick`, tắt & xóa toàn bộ cờ pending trong `NVIC` (`ICER`/`ICPR`), de-init ngoại vi (CAN/UART/DMA) và trả clock về mặc định (HSI):**
   - Nếu không xóa cờ pending NVIC, ngay khoảnh khắc App gọi `__enable_irq()`, NVIC sẽ bắn ngay ngắt cũ còn sót từ thời FBL.
   - Nếu không trả clock từ PLL về mặc định, hàm `SystemInit()` của App cấu hình lại PLL đang chạy có thể bị treo.
3. **Cập nhật thanh ghi bảng vector ngắt `SCB->VTOR = APP_ADDR`** (kèm `__DSB(); __ISB();`):
   - Nếu quên cập nhật `VTOR`, code polling trong App vẫn chạy bình thường, nhưng **ngay khi ngắt đầu tiên nổ ra**, CPU sẽ nhảy vào bảng vector của FBL (`0x08000000`) → ghi đè RAM App hoặc HardFault.
4. **Cập nhật `MSP = app_sp` (`*(uint32_t *)APP_ADDR`) rồi nhảy ngay sang `app_reset` (`*(uint32_t *)(APP_ADDR + 4)`):**
   - Nếu quên đổi `MSP`, App chạy bằng stack cũ của FBL → đâm vào vùng `.bss` của App và phá hỏng dữ liệu/stack.
   - Hai lệnh đổi `MSP` và nhảy `app_reset` nên đặt sát nhau (tốt nhất bằng inline assembly) để tránh đọc biến cục bộ `app_reset` từ stack sau khi vừa đổi `MSP`.
5. **App tự gọi `__enable_irq()` sau khi khởi tạo xong:** Nhảy hàm không reset thanh ghi `PRIMASK`, nên ngắt vẫn tắt suốt quá trình `Reset_Handler` của App cho tới khi App chủ động bật lại.

```c
#define APP_ADDR 0x08008000UL

__attribute__((noreturn))
void fbl_jump_to_app(void)
{
    uint32_t app_sp    = *(volatile uint32_t *)(APP_ADDR);
    uint32_t app_reset = *(volatile uint32_t *)(APP_ADDR + 4UL);

    __disable_irq();

    /* 1. Dừng SysTick */
    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL  = 0;

    /* 2. Tắt và xóa toàn bộ cờ pending NVIC */
    for (uint32_t i = 0; i < 8; i++) {
        NVIC->ICER[i] = 0xFFFFFFFFUL;
        NVIC->ICPR[i] = 0xFFFFFFFFUL;
    }

    /* 3. De-init ngoại vi (CAN/UART/DMA) và trả clock về HSI */
    fbl_deinit_peripherals_and_clock();

    /* 4. Trỏ Vector Table sang Application */
    SCB->VTOR = APP_ADDR;
    __DSB();
    __ISB();

    /* 5. Nạp MSP và nhảy sang Reset_Handler của App bằng assembly */
    __asm volatile (
        "msr msp, %0 \n"
        "bx  %1      \n"
        :
        : "r" (app_sp), "r" (app_reset)
        : "memory"
    );

    for (;;) {}
}
```

---

## 4. Giao thức nạp Firmware (UDS ISO 14229 & CanTp ISO 15765-2)

### 4.1 Phân mảnh gói tin với CanTp (ISO 15765-2)
- Khung Classic CAN chỉ chở tối đa **8 byte** (CAN-FD tối đa **64 byte**), trong khi firmware nặng hàng MB.
- **CanTp (ISO-TP)** đảm nhiệm việc chia nhỏ (segmentation) thành Single/First/Consecutive Frames, kiểm soát luồng (Flow Control — `BS`, `STmin`) và ghép lại ở phía nhận.

### 4.2 Chuỗi dịch vụ nạp chuẩn UDS (ISO 14229)

| Bước | Mã UDS | Tên dịch vụ | Mục đích |
|---|---|---|---|
| 1 | `10 03` | Extended Diagnostic Session | Mở phiên chẩn đoán mở rộng trên App |
| 2 | `27` | Security Access *(nếu yêu cầu)* | Mở khóa Seed/Key trên App |
| 3 | `28` | Communication Control | Tắt lưu lượng CAN bình thường để giảm nhiễu bus |
| 4 | `85` | Control DTC Setting | Tắt ghi mã lỗi (DTC) trong lúc nạp |
| 5 | `10 02` | Programming Session | Ghi cờ `noinit` RAM → **Soft Reset ECU vào FBL** |
| 6 | `27` | Security Access (trong FBL) | Xác thực quyền nạp (Seed/Key) trên FBL |
| 7 | `34/36/37` | *(Tùy chọn)* Tải HIS Flash Driver | Nạp Flash Driver vào RAM |
| 8 | `31` | Routine Control (Erase Memory) | Xóa các sector Flash đích của App |
| 9 | `34` | Request Download | Khai báo địa chỉ đích & kích thước firmware |
| 10 | `36` | Transfer Data | Truyền từng block kèm **Block Sequence Counter (BSC)** |
| 11 | `37` | Request Transfer Exit | Kết thúc truyền dữ liệu |
| 12 | `31` | Routine Control (Check Memory / Dependencies) | Kiểm tra CRC, chữ ký số, chống downgrade, đánh dấu hợp lệ |
| 13 | `11` | ECU Reset | Reset ECU để boot vào App mới |

### 4.3 Xử lý sự cố thực chiến trên UDS
- **Thao tác Flash chậm hơn timeout `P2` (~50 ms):** Khi đang chạy `31` (Erase) hoặc `36` (Write), FBL phải chủ động trả **`NRC 0x78` (Response Pending)** để kéo dài thời gian chờ của tester sang **`P2*`** (thường 5000 ms), kết hợp Tester Present (`3E 80`) để giữ phiên không bị rớt.
- **Trùng Block Sequence Counter (`36` Retry):** Nếu FBL đã ghi xong block #7 và gửi `76 07` nhưng gói phản hồi bị mất trên bus khiến tester gửi lại `36 07`:
  - **Tuyệt đối KHÔNG ghi đè lại vào Flash** (vi phạm ràng buộc Flash / gây lỗi ECC).
  - Theo ISO 14229, FBL bỏ qua payload trùng và trả ngay **Positive Response (`76 07`)** để tester đi tiếp sang block #8.
  - Nếu BSC nhảy sai thứ tự (ví dụ đang chờ `08` mà nhận `0A`), FBL từ chối bằng **`NRC 0x73` (`wrongBlockSequenceCounter`)**.

---

## 5. An toàn mất điện, toàn vẹn dữ liệu & Rollback

### 5.1 Single-Bank vs. Dual-Bank (A/B)

| Tiêu chí | Single-Bank (1 FBL + 1 App) | Dual-Bank (1 FBL + Bank A + Bank B) |
|---|---|---|
| **Dung lượng Flash** | Tiết kiệm (gấp đôi không gian cho App) | Tốn gấp đôi Flash cho App |
| **Mất điện ở 50% khi nạp** | Không chạy được App cũ, **nhưng không brick** (FBL vẫn nguyên vẹn, đứng chờ nạp lại qua UDS) | Tự chạy lại **Bank cũ bình thường** |
| **Phù hợp với** | Nạp ở xưởng dịch vụ / nhà máy | **OTA (Over-The-Air)** ngoài đường |

### 5.2 Cơ chế nguyên tử cho Metadata & Dual-Bank Rollback
- **Phát hiện ghi dở & hỏng bit:** Mỗi bản firmware/metadata luôn đi kèm:
  1. `SW Version` / `Sequence Counter` tăng dần (để chọn bản mới hơn khi cả A và B cùng nguyên vẹn).
  2. `CRC32` hoặc `SHA-256` tính trên toàn bộ payload (phát hiện hỏng dù chỉ 1 bit do brownout).
  3. `Commit / Valid Flag` được ghi **ở bước cuối cùng** sau khi toàn bộ dữ liệu và CRC đã hoàn tất.
- **Tự động Rollback khi App mới bị bug crash (Trial & Confirm):**
  1. Sau khi Bank B nạp xong và pass CRC/chữ ký, FBL đặt Bank B ở trạng thái **`TRIAL / PENDING`** kèm bộ đếm thử (ví dụ `max_tries = 3`) rồi boot sang Bank B.
  2. Nếu Bank B gặp bug logic và bị **Watchdog reset** liên tục mà chưa kịp xác nhận, mỗi lần boot FBL giảm `retry_counter` (`3 → 2 → 1 → 0`) → về `0` là FBL **tự động rollback về Bank A**.
  3. Nếu Bank B chạy ổn định và vượt qua self-test, **chính App ở Bank B** mới ghi cờ **`CONFIRMED`** để chốt bản mới. *(FBL tuyệt đối không tự ghi `CONFIRMED` trước khi nhảy, vì làm vậy sẽ mất hoàn toàn khả năng rollback nếu Bank B chết ngay sau khi boot.)*

---

## 6. Secure Boot, Chữ ký số & HSM

### 6.1 Phân biệt Chữ ký số (Signature) và Mã hóa (Encryption)

| Cơ chế | Mục đích bảo mật | Bắt buộc để chống chạy code giả? |
|---|---|---|
| **Chữ ký số (RSA / ECDSA + SHA-256)** | **Tính xác thực (Authenticity) & Toàn vẹn (Integrity)** — chứng minh đúng OEM phát hành và không bị sửa 1 bit nào | **BẮT BUỘC** |
| **Mã hóa (AES)** | **Tính bí mật (Confidentiality)** — chống đọc ngược (reverse-engineering) tài sản trí tuệ (IP) | Tùy chọn *(mã hóa đơn thuần không ngăn được sửa bit hoặc kẻ tấn công tự mã hóa nếu lộ key)* |

### 6.2 Public Key vs. Private Key / Symmetric Key
- **Private Key** chỉ nằm an toàn trên máy chủ HSM offline của OEM để ký firmware.
- Trên hàng triệu ECU xuất xưởng chỉ lưu **Public Key** (chỉ có quyền *kiểm tra*, không thể *ký giả*).
- Tuyệt đối không dùng chung một Symmetric Key hoặc lưu Private Key trong ECU, vì kẻ tấn công chỉ cần trích xuất key từ **1 ECU duy nhất** là có thể giả mạo firmware cho toàn bộ các xe còn lại.

### 6.3 Chống hạ cấp (Anti-Rollback / Downgrade Attack)
- Kẻ tấn công có thể lấy một bản firmware **v1.0 thật** (có chữ ký OEM hợp lệ 100%, nhưng tồn tại lỗ hổng bảo mật cũ) nạp đè lên xe đang chạy v2.0.
- Kiểm tra chữ ký số **không chặn được** trường hợp này → bắt buộc phải có **Anti-Rollback Security Counter** (bộ đếm tăng một chiều).
- Bộ đếm này phải lưu trong vùng phần cứng không thể bị host CPU xóa/giảm: **HSM exclusive DFLASH** hoặc **OTP / eFuse**.

### 6.4 Secure Boot mỗi lần khởi động (Chain of Trust & HSM CMAC)
- Kiểm tra chữ ký chỉ lúc nạp UDS là **chưa đủ**, vì kẻ tấn công có thể sửa trực tiếp Flash không đi qua UDS (qua debugger DAP/JTAG, tháo chip Flash ngoài, hoặc exploit lúc App đang chạy).
- **Mô hình HSM thực chiến (như trên TC397):**
  - *Lúc nạp UDS:* Kiểm tra chữ ký số (RSA/ECDSA) → nếu hợp lệ, HSM tính và lưu **Golden CMAC (AES-CMAC)** vào vùng nhớ bảo vệ riêng của HSM.
  - *Mỗi lần bật nguồn (Boot):* HSM tính lại **CMAC** rất nhanh trên vùng FBL & App, so sánh với Golden CMAC — khớp mới cho phép boot, sai thì chặn/kẹt lại không cho chạy App.

---

## 7. Bảng tra cứu nhanh các lỗi thực tế hay gặp

| Triệu chứng | Nguyên nhân gốc | Cách khắc phục |
|---|---|---|
| App chạy vòng lặp polling/LED bình thường, nhưng **vừa nổ ngắt đầu tiên là crash / nhảy về FBL** | Quên cập nhật `SCB->VTOR = APP_ADDR` | Cập nhật `SCB->VTOR` + `__DSB(); __ISB();` trước khi nhảy |
| ECU bị **đứng / BusFault** ngay khi chạy UDS `31` (Erase) trên single-bank Flash | CPU đang fetch lệnh từ chính bank Flash đang bị xóa | Nạp và chạy **Flash Driver từ RAM** |
| App vào `main()` thì **không có ngắt nào hoạt động** | FBL gọi `__disable_irq()` nhưng App quên bật lại ngắt | App gọi `__enable_irq()` sau khi khởi tạo xong driver |
| `SystemInit()` của App bị **treo khi cấu hình PLL** | FBL để nguyên PLL đang chạy mà không de-init clock trước khi nhảy | Trả clock về mặc định (HSI) & de-init ngoại vi trước khi nhảy |
| UDS `36` bị báo lỗi ECC / hỏng Flash khi mạng CAN nhiễu và tester retry | FBL ghi đè lại block có cùng số BSC xuống Flash | Bỏ qua payload trùng BSC và trả thẳng **Positive Response (`76 <BSC>`)** |
| ECU bị **brick hoàn toàn** sau khi mất điện / lỗi con trỏ ghi đè sector 0 | Sector 0 bị xóa về `0xFF` (`SP`/`PC = 0xFFFFFFFF` → Lockup) | Khóa vùng FBL bằng **WRP + khóa vĩnh viễn (OTP / RDP2 / HSM)** |
