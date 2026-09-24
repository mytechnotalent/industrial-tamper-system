# OPERATION IRON WEB - Hardware Parts and Products

---
**LEGAL DISCLAIMER:**
The information, tools, and code provided in this repository and course are strictly for educational, research, and defensive purposes only.

You are explicitly prohibited from using any materials contained herein to access, test, modify, or exploit any device, network, or system that you do not own 100% or for which you do not have explicit, documented, and legally binding authorization to interact with.

By using this repository and course, you acknowledge and agree that:
1. Any illegal, unauthorized, or malicious use of this information is solely your responsibility.
2. The author(s) and contributor(s) of this repository and course shall not be held liable for any damages, legal repercussions, criminal charges, or unauthorized actions resulting from the use, misuse, or abuse of the contents herein.
3. You will comply with all applicable local, state, national, and international laws regarding cybersecurity and computer fraud.

**IF YOU DO NOT AGREE WITH THESE TERMS, DO NOT USE THIS REPOSITORY AND COURSE.**
---

The hardware below builds the OPERATION IRON WEB industrial tamper system and its
classroom labs. It is the same breadboard as Acts I to IV, so every peripheral in
the Embedded Hacking kit is **required**: the servo is the shutter latch actuator,
the button is the manual arm request, the infrared eye is the local arm/disarm
remote (ARM, DISARM, and CLEAR), the LEDs annunciate INTRUSION, ARMED, and
SECURE, the DHT11 is the cabinet temperature sensor, the 1602 LCD is the tamper
state, zone, and infection readout, and the RYLR998 radio carries the sealed
tamper command and the mesh link that the labs protect.

The malware track needs one more thing: a **Debug Probe**. It is how you read the
reserved-sector infection marker, watch the `IRONWEB` frame propagate, inject the
magic on the raw payload, and step past the anti-debug trap, so it is effectively
required for Lab 3.

## Radio count at a glance

| Goal | Radios needed | Parts |
| ---- | ------------- | ----- |
| Legitimate sealed tamper command loop | **2** | 1x tamper-node RYLR998 (on the Pico) + 1x mesh gateway RYLR998 (USB-to-TTL) |
| Live attack lab (watch a forged or replayed command land, then die) | **3** | the 2 above + 1x attacker RYLR998 (USB-to-TTL) |
| Worm propagation demonstration | **2 or 0** | watch the firmware emit the `IRONWEB` frame, run a second node, or run the native unit tests |
| Attack lab without a 3rd radio | 2 or 0 | use the offline parser demo or the native unit tests |

A radio never receives its own transmission, and the gateway radio is busy
listening as `gateway.py`, so the live attack needs a separate attacker radio.
The worm propagation lab is observable in the firmware transmit path and in the
native tests, because a single node emits the frame it would hand to its
neighbor.

## Required parts

### Microcontroller and debug

- [1x Raspberry Pi Pico 2 with pre-soldered header](https://www.amazon.com/s?k=raspberry+pi+pico+2+with+pre-soldered+header)
- [1x Raspberry Pi Pico Debug Probe (required for the malware track)](https://www.amazon.com/s?k=raspberry+pi+pico+2+debug+probe)
- [2x USB A-male to micro-USB cable (1 for the Pico 2, 1 for the Debug Probe)](https://www.amazon.com/s?k=micro+usb+cable)

### Breadboard and wiring

- [1x Full-size breadboard (long)](https://www.amazon.com/s?k=full+size+breadboard)
- [1x Assorted jumper wires (male-to-male, male-to-female, female-to-female)](https://www.amazon.com/s?k=breadboard+jumper+wires+assortment)

### Human interface and actuators

- [1x 1602 LCD with PCF8574 I2C backpack](https://www.amazon.com/s?k=1602+lcd+i2c+module)
- [1x DHT11 temperature and humidity sensor (cabinet temperature sensor)](https://www.amazon.com/s?k=dht11+temperature+and+humidity+sensor)
- [1x 10K resistor (DHT11 pull-up, only if your module has none)](https://www.amazon.com/s?k=10k+resistor+assortment)
- [3x 5mm LEDs (1 red, 1 green, 1 yellow)](https://www.amazon.com/s?k=5mm+led+kit)
- [3x 100, 220, or 330 Ohm resistors (for the LEDs)](https://www.amazon.com/s?k=resistor+assortment+kit)
- [1x Push button (tactile switch, the manual arm request)](https://www.amazon.com/s?k=tactile+push+button+assortment)

### Actuator and infrared control surface

- [1x SG90 servo motor (the shutter latch actuator)](https://www.amazon.com/s?k=sg90+micro+servo+motor)
- [1x 1000uF 25V capacitor (servo power stabilization)](https://www.amazon.com/s?k=1000uf+25v+capacitor)
- [1x Infrared (IR) receiver (VS1838B)](https://www.amazon.com/s?k=vs1838b+ir+receiver+module)
- [1x Infrared (IR) remote controller (NEC-compatible local arm/disarm remote)](https://www.amazon.com/s?k=arduino+ir+remote+control)

### LoRa radios and serial adapters

- [3x RYLR998 LoRa module with antenna (tamper node, mesh gateway, and attacker)](https://www.amazon.com/s?k=rylr998+lora+module)
  - Use **2** for the legitimate command loop and **3** for the live attack lab.
  - Use the **same band variant** on every module (for example 915 MHz or 868 MHz).
- [2x USB-to-TTL serial adapter, 3.3V logic (FTDI FT232, CP2102, or CH340)](https://www.amazon.com/s?k=usb+to+ttl+serial+adapter+3.3v)
  - One adapter is the **gateway**. The second adapter is the **attacker** for the live lab.
  - Choose a 3.3V-logic adapter; the RYLR998 is **not** 5V tolerant.
- [2x USB A-male to mini/micro-USB cable for the serial adapters (match your adapter)](https://www.amazon.com/s?k=usb+to+ttl+cable)

## Pin map

| Peripheral | GPIO | Notes |
| ---------- | ---- | ----- |
| DHT11 cabinet temperature sensor | GP4 | 10K pull-up required |
| 1602 LCD SDA | GP2 | I2C1 |
| 1602 LCD SCL | GP3 | I2C1, address 0x27 |
| RYLR998 TX (Pico RX) | GP9 | UART1 |
| RYLR998 RX (Pico TX) | GP8 | UART1 |
| Infrared local arm/disarm remote | GP5 | VS1838B, active low, ARM/DISARM/CLEAR |
| Shutter latch servo signal | GP14 | PWM, 50 Hz, 1000uF bulk on the 5V rail |
| Red INTRUSION LED | GP16 | 220-330 Ohm to ground |
| Yellow ARMED LED | GP17 | 220-330 Ohm to ground |
| Green SECURE LED | GP18 | 220-330 Ohm to ground |
| Manual arm/disarm button | GP15 | Active low, internal pull-up, requests authorization |
| Onboard heartbeat LED | GP25 | Heartbeat |
| Debug Probe | GP0/GP1 | SWD and UART0 console for Lab 3 |

## Cryptography

The authentication layer is implemented entirely in-repo and has no third-party
dependencies:

- **Argon2id** (RFC 9106) derives the 256-bit field key from a provisioned
  passphrase and salt. The classroom profile is `t=3, p=1, m=64` blocks so it
  fits the RP2350 SRAM budget; raise it on the mesh gateway.
- **XChaCha20-Poly1305** seals every request and command with a 192-bit nonce and
  a 128-bit Poly1305 tag, so a forged or modified frame fails authentication and
  never moves the latch.

Act V adds stateful security on top of the sealed wire:

- **Anti-replay window.** A monotonic sequence rule (`last_seq`) rejects a
  captured command on second use, even when its tag is valid.
- **Authenticated state tag.** A keyed tag over the authorization record means a
  debugger that flips the SRAM `granted` boolean is rejected before the latch
  moves.
- **Guarded tamper command set and bounded zone band.** The only accepted
  commands are `TAMPER_COMMAND_ALERT` (`0x01`), `TAMPER_COMMAND_ARM` (`0x02`),
  and `TAMPER_COMMAND_SECURE` (`0x03`), and the only accepted zones are in the
  `0` to `16` band.

The RP2350 has no hardware AES engine (it accelerates SHA-256 only), so ChaCha20
is both the modern and the faster choice on this silicon. The single field key is
derived from the committed lab secret; the design names the key lifecycle
(derive, provision, use, rotate, retire) so a production build can provision key
material from one-time-programmable (OTP) memory and keep a device-unique secret.

## The malware track

The FROSTLINE implant is compiled only under `SANDBOX_ONLY`. It erases and
programs a one-time `0xC7` marker into the reserved sector `0x103FF000` through
the Pico SDK flash API on first run, re-installs on
every later boot, watches the raw inbound frame for the 7-byte magic `IRONWEB`,
and, every 4 ticks while infected and unobserved, emits its 11-byte progress
frame (the magic, the marker `0xC7`, the tick counter, and a probe flag) so a
neighboring node on the classroom mesh that receives it infects itself and emits
it again. It checks CoreDebug `DHCSR` at `0xE000EDF0` to hide from a probe. It is
real in technique and inert in effect: it touches only its own pins, its own
radio frame, and its reserved sector on the same chip, on your own breadboard,
with no network and no host impact. It propagates only across the student's own
breadboard nodes on the classroom network id.
