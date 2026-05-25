# LORAWAN_NODE

Node LoRaWAN end-device chạy trên vi điều khiển STM32F401xC (Cortex-M4), tích hợp stack LoRaMac-node của Semtech. Node đo nồng độ khí CO bằng cảm biến MQ-7, hiển thị kết quả lên màn hình OLED, và gửi dữ liệu lên mạng LoRaWAN theo chu kỳ qua giao thức OTAA.

Cấu hình mặc định:
- MCU: STM32F401xC (Cortex-M4)
- Build system: CMake + Ninja
- Toolchain: GNU Arm Embedded (arm-none-eabi)
- LoRaWAN region: AS923
- Radio driver: SX1276
- Secure element: SOFT\_SE (AES/CMAC bằng phần mềm)

## Phần cứng

| Linh kiện | Kết nối |
|---|---|
| Radio SX1276 | SPI1 — NSS: PA4, RESET: PB0, DIO0: PA3, DIO1: PA8 |
| Cảm biến MQ-7 | Analog — PA0 (ADC1 CH0) |
| Màn hình OLED | I2C1 — địa chỉ 0x3C (SSD1306) |
| LED trạng thái | PC13 (active low, onboard BlackPill) |

## Cấu trúc thư mục

```
Core/Src/
├── main.c          — vòng lặp chính: đọc MQ-7, hiển thị OLED, gọi LoRaWAN_Process()
├── lora_app.c      — state machine LoRaWAN: Join → Wait → Send (mỗi 20 s)
├── oled.c          — driver màn hình SSD1306 qua I2C
├── gpio.c          — khởi tạo GPIO (sinh bởi CubeMX)
├── spi.c           — khởi tạo SPI1 cho radio
└── i2c.c           — khởi tạo I2C1 cho OLED

Core/Inc/
├── Commissioning.h — DevEUI / JoinEUI / AppKey / Region  
├── board.h         — mapping chân GPIO của radio
└── ...             — header HAL sinh bởi CubeMX

Core/LoRaMac/
├── board/          — board port: GPIO, SPI, timer (TIM2 thay RTC), delay
├── mac/            — LoRaMac stack core + region (AS923, EU868, US915, ...)
├── radio/sx1276/   — driver chip radio SX1276
└── peripherals/soft-se/ — mã hóa AES-128 + CMAC bằng phần mềm

cmake/              — toolchain file GCC/Clang, tích hợp STM32CubeMX
CMakeLists.txt      — cấu hình build: chọn radio, region, secure element
CMakePresets.json   — preset Debug / Release
STM32F401XX_FLASH.ld — linker script
```

## Thứ tự chạy

```
startup_stm32f401xc.s       reset handler → gọi main()
└── main()
      ├── MX_GPIO_Init()    cấu hình GPIO
      ├── MX_SPI1_Init()    SPI cho SX1276
      ├── MX_I2C1_Init()    I2C cho OLED
      ├── MQ7_InitAdc()     cấu hình ADC đọc MQ-7
      ├── OLED_Init()       khởi động màn hình
      └── LoRaWAN_Init()    khởi động LoRaMac, cấu hình keys, gửi Join Request

      while(1) — mỗi 500 ms:
      ├── MQ7_ReadRaw()         đọc ADC
      ├── MQ7_AdcToPpm()        tính ppm CO
      ├── LoRaWAN_SetMQ7Raw()   cập nhật payload
      ├── OLED_Update()         hiển thị ADC / Volt / PPM / trạng thái join
      └── LoRaWAN_Process()     chạy state machine LoRaWAN
                JOIN  → gửi Join Request
                WAIT  → chờ Join Accept từ gateway
                SEND  → gửi 2 byte ADC raw lên server, lặp lại sau 20 s
```

## Cấu hình LoRaWAN (OTAA)

Chỉnh file `Core/Inc/Commissioning.h` trước khi build:

```c
// 1 = dùng DevEUI tĩnh | 0 = tự derive từ UID chip STM32
#define COMMISSIONING_STATIC_DEVICE_EUI  1

#define COMMISSIONING_DEVICE_EUI  { 0x00, 0x37, 0x00, 0x19, 0x31, 0x38, 0x36, 0x30 }
#define COMMISSIONING_JOIN_EUI    { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 }
#define COMMISSIONING_APP_KEY     { 0x2B, 0x7E, 0x15, 0x16, 0x28, 0xAE, 0xD2, 0xA6, \
                                    0xAB, 0xF7, 0x15, 0x88, 0x09, 0xCF, 0x4F, 0x3C }
```

Để đổi region, chỉnh `CMakeLists.txt`:

```cmake
option(REGION_AS923 "Region AS923" ON)
option(REGION_EU868 "Region EU868" OFF)
```

> Không commit key thật lên repository công khai.

## Yêu cầu

Cài đặt và thêm vào PATH:
- CMake >= 3.22
- Ninja
- GNU Arm Embedded Toolchain (`arm-none-eabi-gcc`, `arm-none-eabi-objcopy`, ...)

Tuỳ chọn:
- VS Code + CMake Tools extension
- STM32CubeProgrammer CLI (để nạp firmware)

## Build

**Cách 1 — CMake Presets (khuyến nghị):**

```bash
cmake --preset Debug
cmake --build --preset Debug
```

```bash
cmake --preset Release
cmake --build --preset Release
```

Output nằm trong `build/Debug/` hoặc `build/Release/`.

**Cách 2 — Thủ công:**

```bash
cmake -S . -B build/Debug -G Ninja \
      -DCMAKE_TOOLCHAIN_FILE=cmake/gcc-arm-none-eabi.cmake \
      -DCMAKE_BUILD_TYPE=Debug
cmake --build build/Debug
```

**Build lại từ đầu:**

```bash
cmake --build --preset Debug --clean-first
```

**Kiểm tra kích thước firmware:**

```bash
arm-none-eabi-size build/Debug/LORAWAN_NODE.elf
```

## Nạp firmware

File ELF sau khi build: `build/Debug/LORAWAN_NODE.elf`

```bash
STM32_Programmer_CLI -c port=SWD -w build/Debug/LORAWAN_NODE.elf -v -rst
```

## Payload uplink

FPort 2, 2 byte, big-endian:

```
Byte 0: MSB của ADC raw 
Byte 1: LSB của ADC raw
```

## Giấy phép

Repository chứa các thành phần từ STM32Cube (STMicroelectronics) và LoRaMac-node (Semtech). Xem file `LICENSE.txt` trong từng thư mục `Drivers/` và `Core/LoRaMac/` trước khi phân phối lại.
