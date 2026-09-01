# BMP585Sensor

A lightweight C++ driver for the Bosch **BMP585** barometric pressure sensor over **SPI**, targeting the Raspberry Pi Pico SDK (RP2040/RP2350).

The driver handles power-up verification, sensor configuration (oversampling, IIR filtering, output data rate, power mode), and raw-to-physical-unit conversion for temperature and pressure.

---

## Features

- SPI register read/write helpers with correct read-bit framing
- Power-up self-check (chip ID, revision ID, NVM status, POR flag)
- Configurable oversampling (OSR) for pressure and temperature
- Configurable digital IIR filter coefficients
- Configurable output data rate (ODR) and power mode
- Single burst read of temperature + pressure with 24-bit sign extension and scaling

---

## Hardware requirements

- A board running the Raspberry Pi Pico SDK (RP2040 or RP2350)
- A Bosch BMP585 breakout wired to one of the board's SPI peripherals
- A free GPIO for chip-select (CS is handled manually in software, not by the SPI peripheral)

### Example wiring (as used in `main.cpp`)

| BMP585 pin | RP2350 GPIO | Function |
|---|---|---|
| SCK  | 22 | SPI0 SCK |
| MOSI | 19 | SPI0 TX |
| MISO | 20 | SPI0 RX |
| CS   | 11 | Software-controlled chip select |

Adjust these to match your PCB.

---

## Project setup

### 1. Prerequisites

- [Pico SDK](https://github.com/raspberrypi/pico-sdk) installed and `PICO_SDK_PATH` set
- CMake and a GCC ARM toolchain (`arm-none-eabi-gcc`)
- Ninja or Make

### 2. Add the driver to your project

Drop `BMP585Sensor.hpp` and `BMP585Sensor.cpp` into your project's source tree, e.g.:

```
your_project/
├── CMakeLists.txt
├── main.cpp
└── drivers/
    ├── BMP585Sensor.hpp
    └── BMP585Sensor.cpp
```

### 3. CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.13)

include(pico_sdk_import.cmake)

project(bmp585_example C CXX ASM)
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

pico_sdk_init()

add_executable(bmp585_example
    main.cpp
    drivers/BMP585Sensor.cpp
)

target_link_libraries(bmp585_example
    pico_stdlib
    hardware_spi
)

pico_add_extra_outputs(bmp585_example)

# Enable USB/UART stdio as needed
pico_enable_stdio_usb(bmp585_example 1)
pico_enable_stdio_uart(bmp585_example 0)
```

### 4. Build

```bash
mkdir build && cd build
cmake ..
make -j4
```

Flash the resulting `.uf2` to your board in BOOTSEL mode.

> **Note:** the driver currently references `spi0` directly inside `BMP585Sensor.cpp` rather than taking an `spi_inst_t*` parameter. If your sensor is on `spi1`, either edit those calls or extend the constructor to accept an SPI instance.

---

## Usage guide

### 1. Configure the SPI peripheral and CS pin in `main()`

The driver expects the caller to own SPI peripheral setup - it only manages chip-select toggling and register access.

```cpp
constexpr uint PIN_MISO = 20;
constexpr uint PIN_MOSI = 19;
constexpr uint PIN_SCK  = 22;
constexpr uint PIN_BMP585_CS = 11;

spi_init(spi0, 500 * 1000); // 500 kHz — safe starting point

gpio_set_function(PIN_MISO, GPIO_FUNC_SPI);
gpio_set_function(PIN_SCK,  GPIO_FUNC_SPI);
gpio_set_function(PIN_MOSI, GPIO_FUNC_SPI);

gpio_init(PIN_BMP585_CS);
gpio_set_dir(PIN_BMP585_CS, GPIO_OUT);
gpio_put(PIN_BMP585_CS, 1); // idle high
```

### 2. Construct and initialise the sensor

```cpp
BMP585Sensor bmp585(PIN_BMP585_CS);

if (!bmp585.init()) {
    printf("Sensor initialisation failed.\n");
    while (true) tight_loop_contents();
}
```

`init()` performs a dummy SPI transaction (to settle the sensor's SPI interface) then calls `powerUpCheck()` internally. It returns `false` if the chip ID, revision ID, NVM status, or power-on-reset flag don't match expectations - check your wiring and SPI mode/speed if this fails.

### 3. Apply measurement configuration

```cpp
bmp585.configure();
```

This puts the sensor into standby, writes oversampling and IIR settings, then starts normal-mode measurement at the configured ODR. See [Configuration settings](#configuration-settings) below to customise this.

### 4. Read measurements

```cpp
BMP585Sensor::Data sensor_data;

while (true) {
    if (bmp585.readData(sensor_data)) {
        printf("Temp: %.2f degC | Pressure: %.2f Pa (%.2f hPa)\n",
               sensor_data.temperature_c,
               sensor_data.pressure_pa,
               sensor_data.pressure_pa / 100.0f);
    }
    sleep_ms(500);
}
```

`readData()` performs a single 7-byte burst SPI transaction starting at the data registers, reconstructs the 24-bit temperature and pressure values, sign-extends temperature, and scales both into physical units.

### Lower-level access

For custom register work, `readRegister()` and `writeRegister()` are public and can be used directly, e.g. to read back a configuration register to confirm a write succeeded:

```cpp
uint8_t odr_readback = bmp585.readRegister(0x37); // REG_BMP585_ODR_CONFIG
```

---

## Configuration settings

All configuration is applied in `configure()`. The relevant enums are defined in `BMP585Sensor.hpp` and can be mixed and matched.

### Oversampling ratio (`OSR`)

Controls measurement noise vs. power/time trade-off. Higher oversampling reduces noise but increases per-sample conversion time.

| Enum | Multiplier |
|---|---|
| `OSR_1X`   | ×1   |
| `OSR_2X`   | ×2   |
| `OSR_4X`   | ×4   |
| `OSR_8X`   | ×8   |
| `OSR_16X`  | ×16  |
| `OSR_32X`  | ×32  |
| `OSR_64X`  | ×64  |
| `OSR_128X` | ×128 |

Set independently for pressure (`osr_p`) and temperature (`osr_t`) in `configure()`. The default in the driver is `OSR_16X` for pressure and `OSR_2X` for temperature — a reasonable balance for flight/altitude-tracking use cases where pressure accuracy matters more than temperature accuracy.

### Digital IIR filter (`IIRFilter`)

Smooths sample-to-sample noise at the cost of response latency to real changes.

| Enum | Filter coefficient |
|---|---|
| `IIR_BYPASS`    | Off |
| `IIR_COEFF_1`   | 1   |
| `IIR_COEFF_3`   | 3   |
| `IIR_COEFF_7`   | 7   |
| `IIR_COEFF_15`  | 15  |
| `IIR_COEFF_31`  | 31  |
| `IIR_COEFF_63`  | 63  |
| `IIR_COEFF_127` | 127 |

Set independently for pressure (`iir_p`) and temperature (`iir_t`). Heavier filtering is generally undesirable for fast dynamic events (e.g. rocket ascent/apogee detection), where you want the raw response with lighter or bypassed filtering.

### Output data rate (`ODR`)

| Enum | Rate |
|---|---|
| `ODR_240_HZ` | 240 Hz |
| `ODR_100_HZ` | 100 Hz |
| `ODR_50_HZ`  | 50 Hz  |
| `ODR_10_HZ`  | 10 Hz  |
| `ODR_1_HZ`   | 1 Hz   |

Higher ODRs give finer time resolution but increase bus traffic and power draw. `configure()` also sets `deep_dis = 1`, disabling deep-sleep between conversions so the configured ODR is met consistently.

### Power mode (`PowerMode`)

| Enum | Behaviour |
|---|---|
| `MODE_STANDBY`    | No measurements taken |
| `MODE_NORMAL`     | Continuous measurement at the configured ODR |
| `MODE_FORCED`     | Single on-demand measurement, then returns to standby |
| `MODE_CONTINUOUS` | Continuous measurement, ODR/timing behaviour per datasheet |

`configure()` always drops to `MODE_STANDBY` before changing OSR/IIR/ODR settings (required by the sensor), then finishes in `MODE_NORMAL`. If you need on-demand single-shot readings instead of a continuous stream, write `MODE_FORCED` to `REG_BMP585_ODR_CONFIG` after configuring OSR/IIR.

### Customising configuration

Edit the local variables at the top of `configure()`:

```cpp
uint8_t osr_p = OSR_16X;
uint8_t osr_t = OSR_2X;
uint8_t iir_p = IIR_COEFF_3;
uint8_t iir_t = IIR_COEFF_1;
uint8_t odr   = ODR_50_HZ;
```

then rebuild. A future improvement would be to expose these as a `Config` struct passed into `configure()` rather than hardcoded locals.

---

## Data conversion reference

- **Temperature:** raw 24-bit two's-complement value, sign-extended, then divided by `65536.0f` to give °C.
- **Pressure:** raw 24-bit unsigned value divided by `64.0f` to give Pa.

These scale factors come from the BMP585 datasheet's fixed-point output format and assume default sensor calibration is applied internally by the chip (the BMP585 outputs pressure/temperature already compensated — no host-side calibration coefficients are needed, unlike older Bosch sensors such as the BMP280).

---

## License

Refer to LICENSE.md