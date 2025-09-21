# ESP32 + MFRC522 (RFID) — Read Card UID with **Custom SPI Pins**
# Tested with ESP32C6

Read a MIFARE Classic card’s UID (a.k.a. NUID) using an ESP32 and an MFRC522 reader, **routing SPI to your own GPIOs**. This project uses the popular [Miguel Balboa MFRC522](https://github.com/miguelbalboa/rfid) Arduino library.

> ✅ Works on ESP32 with custom SPI pin mapping via `SPI.begin(SCK, MISO, MOSI, SS)`  
> ✅ Prints PICC (tag) type and UID in **hex** and **decimal**  
> ✅ Suppresses repeat logs for the same card until it’s removed

---

## Table of Contents

- [What this project does](#what-this-project-does)  
- [Hardware](#hardware)  
- [Wiring](#wiring)  
  - [Your current mapping (as in code)](#your-current-mapping-as-in-code)  
  - [Safe alternative mapping (portable)](#safe-alternative-mapping-portable)  
- [Software Setup](#software-setup)  
  - [Arduino IDE](#arduino-ide)  
  - [PlatformIO (optional)](#platformio-optional)  
- [Usage](#usage)  
- [Sketch overview](#sketch-overview)  
- [Sample output](#sample-output)  
- [Troubleshooting](#troubleshooting)  
- [Advanced tips](#advanced-tips)  
- [Repository structure](#repository-structure)  
- [Credits & license](#credits--license)  
- [Contributing](#contributing)

---

## What this project does

- Initializes the ESP32 SPI host on **custom pins you define**  
- Initializes the MFRC522 reader and waits for a card near the antenna  
- On new card presentation:
  - Prints the **PICC type** (e.g., *MIFARE 1KB*)  
  - Prints the **UID** in both **hex** and **decimal**  
- Remembers the last UID so the serial log doesn’t spam repeats

---

## Hardware

- **ESP32** development board (any common devkit; pin caveats below)  
- **MFRC522** RFID reader module  
- **RFID tag/card** (MIFARE Classic Mini/1K/4K supported by this example)

> ⚠️ **Voltage:** Power the MFRC522 from **3.3 V** (not 5 V). The module is *not* 5 V tolerant.

---

## Wiring

Keep jumpers short; the MFRC522 is sensitive to wiring length and noise.

### Your current mapping (as in code)

> This is what the included sketch uses and is confirmed **working on your board**.

| MFRC522 Pin | ESP32 GPIO |
|-------------|------------|
| SDA / SS    | **GPIO 6** |
| SCK         | **GPIO 3** |
| MOSI        | **GPIO 5** |
| MISO        | **GPIO 4** |
| RST         | **GPIO 7** |
| 3.3V        | **3V3**    |
| GND         | **GND**    |

> ⚠️ **Note on ESP32 variants:** On classic ESP32‑WROOM/DevKitC boards, **GPIOs 6–11** are attached internally to the flash chip and are typically unsafe to use externally. Since this mapping works for your specific board, you can keep it. If you migrate to another ESP32 and experience boot/read issues, switch to the **safe mapping** below.

### Safe alternative mapping (portable)

| MFRC522 Pin | ESP32 GPIO (recommended) |
|-------------|---------------------------|
| SDA / SS    | **GPIO 5**                |
| SCK         | **GPIO 18**               |
| MOSI        | **GPIO 23**               |
| MISO        | **GPIO 19**               |
| RST         | **GPIO 4**                |

Then change the defines to match and call:
```cpp
SPI.begin(/*SCK=*/18, /*MISO=*/19, /*MOSI=*/23, /*SS=*/5);
```

---

## Software Setup

### Arduino IDE

1. Install **Arduino IDE** (2.x recommended).  
2. Install the **ESP32 board support** (Boards Manager → search *esp32* → Espressif).  
3. Install library **MFRC522 by Miguel Balboa** (Library Manager).  
4. Open the sketch (`.ino`).  
5. Select your **ESP32** board and **COM/Serial** port.  
6. Set **Serial Monitor** baud to **115200**.

### PlatformIO (optional)

Create a `platformio.ini` like:

```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
lib_deps =
  miguelbalboa/MFRC522@^1.4.10
```

---

## Usage

1. Wire the MFRC522 to the ESP32 as per the **Wiring** section.  
2. Flash the firmware.  
3. Open the Serial Monitor at **115200** baud.  
4. Place a MIFARE Classic card near the reader.  
5. Observe PICC type and UID printed in **hex** and **decimal**.  
6. The sketch suppresses duplicate logs while the same card remains on the antenna.

---

## Sketch overview

Below is the exact sketch reflected by this repository (with your custom pin mapping):

```cpp
#include <SPI.h>
#include <MFRC522.h>

// Pick safe GPIOs (avoid boot/flash pins)
// Your current mapping:
#define MFRC_SCK   3
#define MFRC_MISO  4
#define MFRC_MOSI  5
#define MFRC_SS    6   // a.k.a. SDA on MFRC522 boards
#define MFRC_RST   7

MFRC522 rfid(MFRC_SS, MFRC_RST);

// Init array that will store new NUID 
byte nuidPICC[4];

void setup() { 
  Serial.begin(115200);
  SPI.begin(MFRC_SCK, MFRC_MISO, MFRC_MOSI, MFRC_SS);
  rfid.PCD_Init(); // Init MFRC522 
  Serial.println(F("This code scan the MIFARE Classsic NUID."));
}
 
void loop() {
  if (!rfid.PICC_IsNewCardPresent()) return;
  if (!rfid.PICC_ReadCardSerial())   return;

  Serial.print(F("PICC type: "));
  MFRC522::PICC_Type piccType = rfid.PICC_GetType(rfid.uid.sak);
  Serial.println(rfid.PICC_GetTypeName(piccType));

  if (piccType != MFRC522::PICC_TYPE_MIFARE_MINI &&  
      piccType != MFRC522::PICC_TYPE_MIFARE_1K &&
      piccType != MFRC522::PICC_TYPE_MIFARE_4K) {
    Serial.println(F("Your tag is not of type MIFARE Classic."));
    goto HALT;
  }

  if (rfid.uid.uidByte[0] != nuidPICC[0] || 
      rfid.uid.uidByte[1] != nuidPICC[1] || 
      rfid.uid.uidByte[2] != nuidPICC[2] || 
      rfid.uid.uidByte[3] != nuidPICC[3]) {
    Serial.println(F("A new card has been detected."));

    for (byte i = 0; i < 4; i++) nuidPICC[i] = rfid.uid.uidByte[i];
   
    Serial.println(F("The NUID tag is:"));
    Serial.print(F("In hex: "));
    printHex(rfid.uid.uidByte, rfid.uid.size);
    Serial.println();
    Serial.print(F("In dec: "));
    printDec(rfid.uid.uidByte, rfid.uid.size);
    Serial.println();
  } else {
    Serial.println(F("Card read previously."));
  }

HALT:
  rfid.PICC_HaltA();
  rfid.PCD_StopCrypto1();
}

void printHex(byte *buffer, byte bufferSize) {
  for (byte i = 0; i < bufferSize; i++) {
    Serial.print(buffer[i] < 0x10 ? " 0" : " ");
    Serial.print(buffer[i], HEX);
  }
}

void printDec(byte *buffer, byte bufferSize) {
  for (byte i = 0; i < bufferSize; i++) {
    Serial.print(' ');
    Serial.print(buffer[i], DEC);
  }
}
```

---

## Sample output

```
This code scan the MIFARE Classsic NUID.
PICC type: MIFARE 1KB
A new card has been detected.
The NUID tag is:
In hex:  3A  7C  12  5F
In dec:  58  124  18  95
```

If you keep the same card on the antenna, subsequent loops log:
```
Card read previously.
```

---

## Troubleshooting

- **No output / timeouts:**  
  - Verify **3.3V** power and **GND**.  
  - Confirm **SS/SDA** pin wiring matches `MFRC_SS`.  
  - Keep jumpers short.  
  - Try the **safe mapping** (SCK 18, MISO 19, MOSI 23, SS 5, RST 4).  
  - Add `rfid.PCD_DumpVersionToSerial();` in `setup()` to confirm chip detection.

- **Reader detected, but no card reads:**  
  - Present the card flat and wait a second.  
  - Try another MIFARE Classic tag/card.

- **Garbled Serial Monitor / upload issues:**  
  - Ensure **115200** baud.  
  - If using **GPIO 3** (U0RXD) as SCK causes interference, move SCK to **GPIO 18**.

- **Random resets / won’t boot:**  
  - On classic ESP32s, avoid **GPIOs 6–11** (flash), and strapping pins **0, 2, 12, 15**.  
  - Use the safe mapping shown above.

---

## Advanced tips

- **Increase antenna gain:**
  ```cpp
  rfid.PCD_SetAntennaGain(rfid.RxGain_max);
  ```

- **Tune SPI clock (library-dependent):**
  ```cpp
  SPI.beginTransaction(SPISettings(4000000, MSBFIRST, SPI_MODE0)); // 4 MHz
  // Ensure rfid.PCD_Init() is compatible with your library version
  SPI.endTransaction();
  ```

- **HSPI vs VSPI:**  
  ESP32 also exposes `HSPI`. Some library forks let you pass a custom `SPIClass` to `MFRC522`. Otherwise, the global `SPI` instance is used.

---

## Repository structure

```
.
├── README.md
└── ESP32_MFRC522_RFID.ino 
```

---

## Credits & license

- Based on the **MFRC522** Arduino library by **Miguel Balboa**.  
- The original example declares **public domain**.  
- You may choose and include a LICENSE for this repository (e.g., **MIT**).

---

## Contributing

PRs welcome. Helpful contributions include pin‑mapping notes for different ESP32 variants, wiring photos, and tested configurations.
