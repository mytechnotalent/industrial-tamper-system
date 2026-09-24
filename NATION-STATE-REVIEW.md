# OPERATION IRON WEB - Nation-State Accuracy Review

**An adversarial, evidence-based audit of the entire project where every claim is
verified by a re-runnable command or explicitly labelled as a limitation.**

***
**LEGAL DISCLAIMER:**
The information, tools, and code provided in this repository and course are strictly for educational, research, and defensive purposes only. 

You are explicitly prohibited from using any materials contained herein to access, test, modify, or exploit any device, network, or system that you do not own 100% or for which you do not have explicit, documented, and legally binding authorization to interact with.

By using this repository and course, you acknowledge and agree that:

1. Any illegal, unauthorized, or malicious use of this information is solely your responsibility.
2. The author(s) and contributor(s) of this repository and course shall not be held liable for any damages, legal repercussions, criminal charges, or unauthorized actions resulting from the use, misuse, or abuse of the contents herein.
3. You will comply with all applicable local, state, national, and international laws regarding cybersecurity and computer fraud.

**IF YOU DO NOT AGREE WITH THESE TERMS, DO NOT USE THIS REPOSITORY AND COURSE.**

***

## 1. Scope and Method

This review treats the project as hostile-to-itself. Every module, constant, test
vector, document, and artifact is independently checked. The method is:

1. **Re-run every gate** (`audit_c_standard`, `audit_python_standard`,
   `run_tests`, `check_coverage`) and record the exact output and exit codes.
2. **Re-verify every constant** against the generated artifact header and the
   JSON source of truth, by regenerating the header and diffing it.
3. **Re-verify every cryptographic claim** against a published standard vector,
   and separate vectors that run in the native C suite from vectors that run only
   in the Python suite.
4. **Audit the worm adversarially**, with a dedicated Worm and Propagation
   section: what it does, how it spreads, how it is detected, how it is bounded
   and removed, and what it does not prove.
5. **Re-verify every artifact** by SHA-256 and by the in-repo build guardrail,
   because this project ships no CTF firmware artifact.
6. **Read the documents adversarially** for overclaims, stale numbers, and
   omissions, then correct them in this review.

## 2. Gate Results (all re-run for this review)

| gate | command | observed result |
|---|---|---|
| C standard | `python3 scripts/audit_c_standard.py` | exit 0, no output, **0 violations** |
| Python standard | `python3 scripts/audit_python_standard.py` | exit 0, no output, **0 violations** |
| Native tests | `python3 scripts/run_tests.py` | **458 checks, 0 failures**, 141 test cases |
| Coverage | `python3 scripts/check_coverage.py` | exit 0, **100.00% line coverage**, 2052 owned lines |
| Python suites | `python3 -m unittest test.test_field_crypto test.test_tamper_node -v` | **17 tests, OK** |

The coverage gate passes on line coverage. It does not require 100% branch or
region coverage, and the raw report is not 100% there: regions 99.30% and
branches 93.84%. That gap is real and is stated in the module table below.

## 3. Module-by-Module Audit

Owned lines are the instrumented statement lines reported by `llvm-cov report`
through `check_coverage.py`. Raw `wc -l` over `src/*.c` includes comments and
blank lines; `main.c` is excluded from coverage by design. Every owned module is
at 100.00% line coverage.

| module | role | owned lines | line coverage | verification performed | honest limitation |
|---|---|---|---|---|---|
| `crc.c` | CRC-16/CCITT-FALSE diagnostic | 20 | 100.00% | `test_crc16_ccitt` check value `0x29B1` | not on the wire; a checksum is not authentication |
| `sensor.c` | DHT11 cabinet-band classifier | 151 | 100.00% | waveform, all timeout shapes, CRC error, climate ok/reject/invalid | mock GPIO replays a recorded waveform, not real silicon |
| `display.c` | HD44780 over PCF8574 | 73 | 100.00% | `test_display_format_lines`, `test_display_render_lines` via recorded I2C | mock I2C, not real HD44780 bus timing |
| `radio.c` | RYLR998 provisioning, `AT+SEND`, `+RCV` parser | 187 | 100.00% | build/parse/reject/pump, oversize guards, `test_radio_spoofed_sender_attribution` | mock UART; the RF band is not simulated |
| `status_led.c` | red/yellow/green INTRUSION annunciator | 14 | 100.00% | `test_status_led_show` | none |
| `button.c` | ARM/debounce input | 34 | 100.00% | pressed/consume/debounce/reset | active-low input is exercised only through mocks |
| `servo.c` | 50 Hz latch PWM | 26 | 100.00% | `test_servo_map`, `test_servo_init`, `test_servo_actuate` | mock PWM; no real servo or inrush load |
| `ir_remote.c` | VS1838B NEC arm/disarm decode | 116 | 100.00% | decode valid/reject/bad leader/mark/ambiguous/address/command, `test_ir_poll_*` | optical path is unauthenticated; no anti-replay |
| `latch.c` | latch state machine and fail-safe policy | 41 | 100.00% | `test_latch_init`, open/close travel, `test_latch_fail_safe`, `test_latch_reject_unauthorized` | bounded travel only; no real shutter or load |
| `control.c` | sealed tamper command path | 96 | 100.00% | `test_control_handle_success`, bad command, bad zone, bad tag, replay, short body, key guards | shared lab key; guarded set is three commands |
| `tamper_auth.c` | anti-replay window and state tag | 74 | 100.00% | `test_tamper_auth_state_tag`, apply window/advance/bad tag, null, key, and tag guards | deterministic nonce from sequence; single key |
| `chacha20.c` | ChaCha20 and HChaCha20 | 99 | 100.00% | RFC 8439 block and stream vectors, HChaCha20 draft vector | none |
| `poly1305.c` | Poly1305 one-time authenticator | 169 | 100.00% | RFC 8439 tag vector, aligned path | none |
| `crypto_aead.c` | XChaCha20-Poly1305 seal/open | 38 | 100.00% | round-trip, tamper tag/ct/ad, constant-time `tag_equal` | built from the in-repo primitives, not an audited library |
| `blake2b.c` | BLAKE2b and Argon2 H' | 161 | 100.00% | `test_blake2b_abc`, multiblock, H' 32 and 256 vectors | none |
| `argon2.c` | Argon2id core (BLAMKA, hybrid addressing) | 336 | 100.00% | `test_argon2_lanes`, `test_argon2_type_i`, `test_argon2_clamp` branch coverage | the RFC 9106 KAT runs in Python, not in this C suite |
| `crypto_kdf.c` | Argon2id field key derivation | 28 | 100.00% | reject, empty password, determinism, salt sensitivity | classroom profile `t=3 p=1 m=64`; committed passphrase and salt |
| `envelope.c` | hex nonce/ciphertext/tag codec | 91 | 100.00% | nonce, round-trip, seal/open rejects, uppercase, known vector | none |
| `monitor.c` | tamper controller state machine | 227 | 100.00% | init, idle, render (including infection), climate, arm/disarm remote, remote arm/alert/secure/replay/bad tag/bad command, arm no-bypass, link loss, link unseen/within, guards, implant frame delivery | mocks are not the real silicon |
| `implant.c` | SANDBOX_ONLY FROSTLINE worm | 71 | 100.00% | first run, re-install on boot, infection marker, debug attached, worm magic match, frame build, propagation flag and gate, handle command, propagate, anti-debug, clean tick | benign educational worm; build-guarded and breadboard-bound |
| `main.c` | entry point | n/a | excluded | build only | excluded from coverage by design |

**Total owned lines at 100.00% line coverage: 2052.**

Branch coverage below 100% in the same report: `display.c` 85.71%, `monitor.c`
85.71%, `radio.c` 89.87%, `control.c` 90.48%, `tamper_auth.c` 92.31%,
`envelope.c` 92.86%, `sensor.c` 94.74%, `ir_remote.c` 96.00%, `argon2.c`
97.56%.

## 4. Cryptographic Claim Verification

The native suite asserts the following published vectors. Each name below appears
as a passing case in the `run_tests.py` output for this review.

| claim | standard | vector | observed |
|---|---|---|---|
| ChaCha20 block function | RFC 8439 section 2.3.2 | key 00..1f, nonce 000000090000004a00000000 | `test_chacha20_block` PASS |
| ChaCha20 stream cipher | RFC 8439 section 2.4.2 | "Ladies and Gentlemen..." 114-byte ciphertext | `test_chacha20_stream` PASS |
| HChaCha20 subkey | XChaCha20 draft (irtf-cfrg-xchacha) | published subkey vector | `test_hchacha20` PASS |
| Poly1305 tag | RFC 8439 section 2.5.2 | "Cryptographic Forum Research Group" tag `a8061dc1305136c6c22b8baf0c0127a9` | `test_poly1305`, `test_poly1305_aligned` PASS |
| BLAKE2b-512 | BLAKE2 reference | digest of "abc", multiblock, long-input | `test_blake2b_abc`, `test_blake2b_multiblock` PASS |
| Argon2 variable-length hash H' | RFC 9106 section 3.3 | H' of {1,2,3,4} at 32 and 256 bytes | `test_blake2b_long_short`, `test_blake2b_long` PASS |
| Argon2id known-answer | RFC 9106 section 5.3 | `0d640df58d78766c08c037a34a8b53c9d01ef0452d75b65eb52520e96b01e659` | `test.test_field_crypto.TestFieldCrypto.test_rfc9106_argon2id_vector` PASS (Python suite) |
| Envelope layout | project vector | known nonce, node id 7, fixed body | `test_envelope_known_vector` PASS |
| Firmware and Python interop | project vector | shared field key and envelope | `test_field_key_matches_firmware`, `test_envelope_matches_firmware` PASS (Python suite) |

The RFC 9106 Argon2id known-answer test is a Python `unittest` in
`test/test_field_crypto.py`; it is not part of the 458 native checks. Running the
Python suites directly confirms all 17 tests pass, including the KAT and the
firmware-interop vectors.

### 4.1 Sealed command path, guarded set, and bounded zone band

`src/control.c` opens the envelope under the field key with the tamper node id as
associated data, then `control_parse` rejects the body unless the command byte is
one of `TAMPER_COMMAND_ALERT` (`0x01`), `TAMPER_COMMAND_ARM` (`0x02`), or
`TAMPER_COMMAND_SECURE` (`0x03`), and the decoded zone lies between
`TAMPER_ZONE_MIN` (`0`) and `TAMPER_ZONE_MAX` (`16`). The guarded set is
therefore exactly those three commands, and the accepted zone band is exactly 0
to 16. The behavior is asserted by `test_control_handle_success`,
`test_control_command_set`, `test_control_authorize`, `test_control_replay`,
`test_control_bad_command`, `test_control_bad_zone`, `test_control_bad_tag`,
`test_control_short_body`, and `test_control_key_guards`. All pass in this
review.

### 4.2 Anti-replay sequence window

`tamper_auth_apply` in `src/tamper_auth.c` accepts a command only when
`seq > auth->last_seq`, then verifies the keyed tag against the candidate record,
then advances the floor. The behavior is asserted by
`test_tamper_auth_apply_window` (accept once, reject the same sequence, reject an
older sequence), `test_tamper_auth_apply_advance` (a newer sequence advances
`last_seq`), `test_tamper_auth_bad_tag`, and the end-to-end
`test_monitor_remote_replay`. All pass in this review.

Honest limitation: `last_seq` is plain SRAM and resets to zero on every boot, so
a command captured before a reboot can be replayed after one. The paper's Threat
Model states this; the README does not. A production controller would persist the
floor in non-volatile memory.

### 4.3 Authenticated state tag

`tamper_auth_state_tag` seals a nine-byte record (`granted`, `seq[4]`,
`last_seq[4]`) under the field key with a nonce built from the sequence and the
domain byte `0xA7`; `tamper_auth_state_ok` recomputes and compares in constant
time (`crypto_aead_tag_equal` is a branchless XOR accumulator).
`test_tamper_auth_state_tag` proves a modified record fails, and the monitor paths
prove the end-to-end denial before the latch moves.

Honest limitations: the lab derives the field key and the state-tag key from one
committed secret, so a compromised device can compute tags the gateway accepts;
the tag protects against casual tamper and a debugger that flips `granted`, not a
physical attacker who can read the key out of SRAM and recompute the tag; there
is no per-device key or rotation; and the nonce is deterministic in the sequence,
so two distinct records that ever share a sequence would violate AEAD nonce
uniqueness.

## 5. Worm and Propagation Audit

Act V is the propagation act, so the worm gets its own dedicated audit. The
review asks six questions: what it does, how it spreads, how it is detected, how
it is bounded and removed, and what it does not prove.

### 5.1 What it does

`src/implant.c` is compiled only under `SANDBOX_ONLY`. The clean firmware build
does not define the guard, so the shipping image has no worm. In the
`SANDBOX_ONLY` build:

- **Reserved-sector infection.** `implant_init` reads the marker byte at
  `TAMPER_IMPLANT_RESERVE_ADDR` (`0x103FF000`). On the first run the marker is
  absent, so `implant_infect` erases the sector and programs `0xC7`
  (`TAMPER_IMPLANT_MARKER_BYTE`) with the Pico SDK flash API exactly once.
  `implant_infected` reports the marker by comparing the reserved byte to
  `0xC7`. The marker is not a simulated memory-mapped store; `flash_range_erase`
  and `flash_range_program` are the only operations that persist on real flash.
- **Re-install on boot.** On every later boot the marker is present, so
  `implant_init` sets the armed latch and the payload handler is live again
  without any firmware change. A reflash of the program region does not touch the
  reserved sector.
- **The IRONWEB frame.** `implant_build_worm` writes an 11-byte frame: the
  7-byte magic `IRONWEB` (`TAMPER_IMPLANT_WORM_MAGIC`), the marker byte `0xC7`,
  the low and high bytes of the tick counter, and a byte that reports whether a
  probe is attached (`implant_debug_attached`).
- **Propagation.** `implant_tick` increments `g_implant_ticks`; while the node is
  infected and no probe is attached, every
  `TAMPER_IMPLANT_PROPAGATE_INTERVAL_TICKS` (4) ticks `implant_propagate` builds
  the frame and emits it over the mesh radio.
- **Raw-frame magic handler.** `implant_handle_command` matches the exact 7-byte
  magic on the inbound payload, arms the payload, writes the marker, and
  propagates immediately. Critically, `monitor_apply_frame` delivers the raw
  payload to this handler before it calls the sealed `control_handle_frame`, so
  the worm matches bytes underneath the authenticated path.
- **Anti-debug.** `implant_debug_attached` reads CoreDebug `DHCSR` at
  `0xE000EDF0` (`TAMPER_IMPLANT_DHCSR_ADDR`). Bit 0 is `C_DEBUGEN` and bit 1 is
  `C_HALT`. When either is set, `implant_tick` and `implant_handle_command`
  return early and both the handler and the propagation are suppressed.

Every one of these behaviors is asserted by a native test:
`test_monitor_implant_frame`, `test_implant_init_first_run`,
`test_implant_reinstall_on_boot`, `test_implant_infected_marker`,
`test_implant_debug_attached`, `test_implant_propagation_flag`,
`test_implant_worm_match`, `test_implant_build_worm`,
`test_implant_propagate_gate`, `test_implant_handle_command`,
`test_implant_propagates`, `test_implant_anti_debug`, and
`test_implant_tick_clean`. The Python adapter additionally asserts the
propagation with `test_09_implant_propagates`. All pass in this review.

### 5.2 How the propagation works, and why one clean node is not enough

The propagation is deliberately narrow. On an infected, unobserved node, every
four ticks, the worm emits one 11-byte `IRONWEB` frame over the same LoRa mesh
link and network id the node uses for its tamper traffic. A peer that receives
the frame and runs the same handler matches the magic, infects itself, and emits
the frame again. The infection is therefore state plus a loop: the state is the
`0xC7` marker in the reserved sector, and the loop is the four-tick emission. A
firmware reflash writes the program region and leaves the marker alone, so the
node comes back infected, and a reflash of one node leaves every already-infected
neighbor untouched. Removing the payload therefore requires three steps that must
all happen: close the propagation gate, erase the marker, and remove the code
path and the build flag. That is enough to defeat a naive remediation, which is
the lesson.

### 5.3 How it stays underneath the authenticated path

The worm is not a stealth protocol client. It never builds an envelope, never
holds a key, and never calls `control_handle_frame`. Its entire inbound surface
is `memcmp` against the 7-byte magic on the raw frame, and its entire outbound
surface is the same radio send path the sensor telemetry uses. That is the
architectural point: the sealed command path can be correct, tested, and
replay-resistant, and a listener that sits below it is unaffected by every one of
those properties. The monitor's ordering makes the lesson explicit in one
function: the raw payload reaches the worm before it is authenticated.

### 5.4 How it is detected and removed

- **By build comparison.** The clean and `SANDBOX_ONLY` images differ by the
  implant translation unit and its symbols, which is the simplest and strongest
  detection: the payload is absent from the shipping build.
- **By reserved-sector inspection.** The marker at `0x103FF000` is state the
  firmware image does not own, and it is visible with the Debug Probe or
  `picotool`.
- **By mesh signature.** The periodic 11-byte `IRONWEB` frame on a four-tick
  cadence is a static and dynamic signature on the radio.
- **By display mismatch.** The `I:INF` infection field on the LCD next to a
  quiet command link is the worm's fingerprint.
- **By static analysis.** The `IRONWEB` magic, the `implant_propagate` emission,
  the `DHCSR` read address, and the reserved-sector address are all literal
  constants in the image.
- **By controlled observation.** Because the anti-debug branch is a single early
  return, a student can break after it under GDB and observe the handler and the
  propagation resume, which proves the payload rather than merely suspecting it.
- **By removal.** The documented fix is three steps that must all happen: close
  the propagation gate with `implant_set_propagation(false)`, erase the reserved
  sector, and remove the re-install check and the `SANDBOX_ONLY` build flag so no
  future boot trusts the marker.

### 5.5 How it is bounded

The worm is bounded by construction and by test. It touches only its own radio
frame, its arming latch, and the one reserved sector. It has no network, no
filesystem, and no host impact. It uses synthetic data only. It never actuates
the latch and never opens the sealed path. The `SANDBOX_ONLY` guard is the
containment boundary, the reserved sector is on the same chip and holds nothing
else, and the native worm tests assert both the behavior and its limits.

### 5.6 What it does not prove, and the honest limitation

The worm is a **benign educational worm**. It is confined to the breadboard,
guarded by `SANDBOX_ONLY`, and has no network. It emits only to the student's own
LoRa modules on the classroom network id, it actuates nothing, and it writes only
to a reserved sector on the same chip that holds nothing else. It spreads only
across the student's own breadboard nodes, and its propagation is a re-emission
over the same classroom radio link, not an internet worm and not a routable
payload. It is a demonstration of technique, not tradecraft: it does not encrypt
itself, it does not load a second stage, it does not exfiltrate anything, it does
not persist across a deliberate sector erase, and it does not resist physical
forensics. Its persistence is persistence against a firmware reflash, and its
propagation is propagation against a network the student owns. Any claim that
this module is operationally representative would be an overclaim, and this
review records that plainly.

The deeper honest limitation is architectural: the worm bypasses the entire
sealed command path by listening to the raw payload before it is authenticated,
and it keeps a copy of itself outside the image. No amount of wire authentication
fixes the first, and no protocol fixes the second. Mitigating them is a
build-integrity, image-signing, network-quarantine, debug-lockdown, and
state-erasure control, which is why the blue half names those controls rather
than pretending the protocol covers them.

## 6. Artifact Verification

This project ships no CTF firmware artifact (`build/` holds only untracked local
test binaries). The companion CTF is external:
`https://github.com/mytechnotalent/CTF_industrial-tamper-system`, which ships the
compromised image with the worm and its verifier. What is verified in this
repository is the source tree and the provisioning artifact.

Source-tree aggregate SHA-256 over all 44 `.c` and `.h` files under `src/` and
`include/`, computed as `find src include ... | sort | xargs shasum -a 256 |
shasum -a 256`:

```
9057bacc34b726712321ca32a33f65c79818e339ae353d04b4a522e4b7e8e777
```

Key artifacts by SHA-256:

```
paper.pdf                     79d875da26d631d28aaa459ec71f2a460aaed8590492cefc0152e34b8c4205e7
paper.typ                     bb0f8e8b81c7ab7c0d0495d6e803c4c7819623115f69bfaf3eff3d1dfab2d842
industrial-tamper-system.png  f323a90532dfac4990b2b3464b12db238b810da039ab921bc09b883bd1201f2d
scripts/packet_artifact.json  ad4b970e3370ba69b3626d1a0795740ec54e4d5d7143f445e29757cd83a34d0b
include/packet_artifact.h     db5ce88fb19eaea026fab73e857f3661054424570de5a72194320c9ecdc055da
include/field_secrets.h       63cffbbeb9e4740c4031865d6bcf5bb8302ede090579eb4fbf0ef41764135d07
```

The build guardrail `check_packet_artifact_header` regenerates
`include/packet_artifact.h` from `scripts/packet_artifact.json` and fails if the
committed header is stale. Re-run for this review:

```
$ python3 scripts/gen_packet.py --from-json scripts/packet_artifact.json \
      --header-out /tmp/iw_pa.h --check-header-path include/packet_artifact.h
Wrote generated firmware header: /tmp/iw_pa.h
Verified header matches: include/packet_artifact.h
exit=0
```

Constants re-read from `include/packet_artifact.h` and matched to the README and
the pin map: `PACKET_NODE_ADDRESS` 7, `PACKET_GATEWAY_ADDRESS` 0x0001,
`PACKET_FRAME_SIZE` 48, `PACKET_ALERT_WAIT_MS` 5000,
`PACKET_LATCH_CLOSE_PULSE_US` 500, `PACKET_LATCH_OPEN_PULSE_US` 1500,
`PACKET_DHT_TIMEOUT_US` 240, `PACKET_LCD_I2C_ADDRESS` 0x27,
`PACKET_MAX_RCV_LEN` 256, plus the shared pin map.

The banner is generated by `scripts/gen_banner.py` and is 1500 x 1500 pixels,
matching the other acts.

## 7. Adversarial Document Review

| document claim | audit verdict |
|---|---|
| README does not imply the device is unhackable | **accurate**; it states the protocol is sealed, that the worm listens underneath it, and that removal is not a single patch |
| README states the worm is benign, guarded, and confined | **accurate**; it appears in the narrative, the malware section, Lab 3, PARTS.md, and this review |
| README states the worm spreads only across the student's own breadboard nodes | **accurate**; it is stated in the narrative, the FROSTLINE Worm section, Lab 3, PARTS.md, and this review |
| README gives the marker, magic, interval, anti-debug, and propagation details | **accurate**; the `0xC7` marker, `IRONWEB` magic, 4-tick interval, 11-byte frame, `DHCSR` bits, and reserved address all match the firmware |
| README states the clean build does not define `SANDBOX_ONLY` | **accurate**; the CMake option defaults to OFF and the module is guarded |
| README "The suite has **141 cases** and **458 checks**" | **accurate**; the native runner reports exactly 141 cases and 458 checks |
| README claims 100% line coverage of owned modules | **accurate**; the report is 2052 / 2052 lines |
| README does not mention per-device key rotation | **omission**; the paper's Threat Model is the only place that states the single-key limitation |
| README anti-replay section does not mention the reboot reset | **omission**; the paper states it, the README does not |
| README propagation section states the worm re-emits to a peer | **accurate**; it matches `implant_tick`, `implant_propagate`, and the raw-payload handler ordering in `monitor_apply_frame` |
| Python tooling command constant | **defect**; `scripts/gateway.py` names `TAMPER_COMMAND_ARM = 1` and sends that byte, while the firmware decodes `0x01` as `TAMPER_COMMAND_ALERT` and `0x02` as `TAMPER_COMMAND_ARM`; the byte is still a guarded command, so the guard is not bypassed, but the tooling and the firmware disagree on the command's meaning |

Corrections: the README's anti-replay and key-model sections should carry the
same reboot-reset and no-per-device-rotation caveats the paper already carries.
The Python command-constant mismatch should be reconciled in the tooling so the
name and the decoded command agree. No claim of unhackability was found.

## 8. Honest Limitations

- **Physical access wins.** A Debug Probe over SWD can read the field key from
  SRAM. The authenticated state tag detects a flipped verdict, but a probe that
  can read the key and recompute the tag defeats the design. Only OTP debug
  disable closes this.
- **Key extraction from flash.** `include/field_secrets.h` commits the passphrase
  and salt. Anyone holding the image holds the key. This is a lab convenience,
  not a deployment.
- **Single shared field key.** Both the wire key and the state-tag key derive
  from one committed secret, so a compromised device can compute tags the gateway
  accepts. There is no per-device key and no rotation in this build.
- **Replay after reboot.** `last_seq` resets to zero, so a command captured
  before a power cycle can be replayed after it. The floor is not persisted.
- **Classroom crypto profile.** Argon2id runs at `t=3 p=1 m=64` to fit SRAM; the
  state-tag nonce is deterministic in the sequence; both are teaching parameters,
  not hardening parameters.
- **The worm is inert, guarded, and breadth-limited.** It is benign,
  breadboard-bound, `SANDBOX_ONLY`-guarded, networkless, and confined to a
  reserved sector on the same chip. Its persistence is persistence against a
  firmware reflash, and its propagation is propagation across the student's own
  breadboard nodes, not against a deliberate sector erase, physical forensics, or
  a real network. It demonstrates technique, not tradecraft.
- **The worm bypasses the protocol.** A module that reads the raw payload and
  re-emits it is a build-integrity and network-containment problem, not a
  wire-authentication problem. Signing, quarantine, and debug lockdown are named
  as the real controls.
- **Unauthenticated optical input.** Any NEC remote can send an arm request. The
  optical surface is a documented exposure; the sealed radio path is the
  authorization path.
- **Manual arm is a single input.** It is debounced and it never bypasses
  authorization, but it is one button; a failed button or a stuck line is a
  hardware reliability problem outside the firmware's control.
- **Supply chain and sensor trust are out of scope.** The DHT11 is checksummed,
  not authenticated, and the firmware is only as trustworthy as the toolchain and
  the parts.
- **Availability is not protected.** An attacker on the band can jam or flood the
  receiver.
- **Coverage is line coverage.** Branch coverage is not 100%, and the harness
  mocks are not the real silicon.

## 9. Conclusion

The project is internally consistent and candid: 20 owned modules, 2052
instrumented lines, 100.00% line coverage, 458 native checks passing with 0
failures, 141 native cases, and 17 passing Python tests, every cryptographic
primitive anchored to a published vector. The four gates all pass with exit 0.
Act V adds real propagation and infection behavior over Acts I to IV:
raw-payload magic matching beneath the authenticated command path, a reserved-
sector infection marker that survives a firmware reflash, a re-install on every
boot, a four-tick propagation loop that makes one node a source for the next, and
a three-step removal that closes the gate, erases the sector, and removes the
code path. It keeps the sealed and guarded tamper command path, the strictly
monotonic anti-replay window, and the keyed tag over the authorization record,
all exercised end to end, and it adds an arm request that asks for authorization
instead of bypassing it. Every worm behavior is asserted by a native test and
every limit is stated. The documentation is unusually honest about the shared
key, the open debug port, the inert worm, and the scope limit that it spreads
only across the student's own breadboard nodes, with minor omissions (reboot
reset and no per-device rotation) that should be folded into the README, and one
tooling command-constant mismatch that should be reconciled. The core lesson
holds and is stated: the wire is sealed, the verdict is tagged, the arm request
cannot bypass, and the remaining risk is the key, the probe, the copy of the
payload that the reflash never touched, and the copy on the next board.

---

*This review is reproducible: run the five commands in section 2, the header
check in section 6, and the Python suites in section 4.*

This is Act V of the ten-act OPERATION COLD IRON saga. See SAGA.md.
