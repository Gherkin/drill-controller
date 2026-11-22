# Micro drill press controller — plan

Closed-loop spindle controller for a 20k rpm micro-drilling press. RPM is the control variable, load torque is compensated as fast as possible, and the controller actively protects the bits.

---

## 1. Goals

- Hold the RPM setpoint under varying drilling torque, reacting within about 1 ms.
- Prevent micro bits from snapping: monitor chip load and torque.
- Measure table height for depth readout and feed-rate sensing.
- Local UI: 2.0" SPI TFT plus a rotary encoder.

## 2. Open questions (answer before schematic)

- [x] Motor voltage: 24 V
- [x] Motor terminal resistance: R ≈ 4.66 Ω (lowest of 4 multimeter readings 4.74–7.42 Ω, minus 0.083 Ω probe resistance, 2026-11-21). Stall current ≈ 24 V / 4.66 Ω ≈ 5.2 A (cold winding)
- [ ] Motor no-load and drilling current, and inductance L. Deferred: not safe to run the motor loose on the bench. Measure in the press via IPROPI logging (phase 1)
- [x] Rear shaft: Ø5 mm, about 8 mm free
- [x] Table: lever + link (not rack and pinion), table on vertical guide rods, central post through the base, about 50 mm travel
- [ ] Bit range to support (e.g. 0.2–1.0 mm)
- [x] Supply: external 24 V PSU, 5.5×2.5 mm plug, centre positive. Not bought yet; buy a regulated 24 V brick of at least 6 A (150 W) whose output stays ≤ 25 V and that has short-circuit/overcurrent protection (hiccup or foldback); the board has no input fuse (#38). A lab supply must have its current limit set to 6 A or less. Bench: plug into J1 on the board. Final install: panel-mount jack on the front, wired to the J11 screw terminal

## 3. System architecture

```
 AS5047P (spindle angle, ABI) ──┐
 Table height sensor ───────────┼──> STM32G4 ──> Motor driver ──> Spindle motor
 Rotary encoder (+ button) ─────┘       │   ^          │
                                        │   └──────────┘ current sense (IPROPI / shunt)
                                        └──> TFT display (SPI1, shared with AS5047P)
                                        └──> Buzzer
```

## 4. Hardware blocks

| Block | Choice | Notes |
|---|---|---|
| MCU | STM32G431CBT6, LQFP-48 (LCSC C529355, Extended) | Advanced timer for PWM, timer in encoder mode for ABI, ADC triggered at PWM centre, CORDIC/FMAC. STM32G474 is the alternative (datasheet D62) |
| Spindle sensor | AS5047P-ATSM (LCSC C962063, Extended) + 6×2.5 mm diametric NdFeB | On-axis, 1–3 mm gap, centred within ±0.25 mm. Use ABI at 1000 counts/rev (edge-rate limit at 20k rpm) and SPI for diagnostics |
| Motor driver | DRV8874PWPR (LCSC C1855818, Extended) | Stall ≈ 5.2 A is below the 6 A peak rating (D56 p4) and the 6 A minimum OCP trip (D56 p6). Set the VREF hardware current limit below 6 A. Integrated IPROPI current sense. DRV8876 (3.5 A peak) is ruled out; DRV8873 (8 A, D58 p17) is the fallback. EasyEDA footprint fixed (#15): the 4 thermal vias are pad 17 (EP, GND), tented, no paste (D56 p40–41). It still has 4 vias (0.3 mm drill) where TI's example has 12 (0.2 mm), and 1.75 × 0.34 mm lead pads where TI's example has 1.5 × 0.45 mm (D56 p40). Bring-up test pads (not fitted by JLC, #36): TP13 IPROPI, TP14 DRV_VREF, TP15 MOTOR_A (OUT1), TP16 MOTOR_B (OUT2) |
| Current sense | IPROPI mirror, or shunt + amplifier | Sampled centre-aligned with the PWM |
| PWM | 20–40 kHz | Inaudible. Check ripple against motor L/R |
| Table height | AS5311-ATST (LCSC C22459687, Extended, low stock) + ~60 mm magnetic strip on the table's central post, sensor fixed inside the base | Measures the table directly, so no linkage nonlinearity or pin play. Incremental over 50 mm: zero on contact or on the bottom stop. Fallback: caliper scale in the base, or an angle sensor on the lever pivot with a calibrated lookup table |
| Spindle magnet mount | Turned aluminium/brass cap, press-fit onto the Ø5 shaft (~5 mm engagement), Ø6×2.5 diametric magnet in a pocket | Avoid a printed cap at 20k rpm (creep, imbalance). Sensor PCB on a bracket on the head, 1–3 mm gap |
| Power | Two inputs in parallel on VIN_RAW, both before the reverse-polarity FET, so either one is protected. J1 XKB DC-005-5A-2.5 DC jack (C381115, Extended; 2.5 mm pin, 57 V 5 A, D78 p1) for bench use; pin 1 = centre (+), pins 2 and 3 (sleeve and switch) to GND. J11 Xinlaiya XY128V-A-5.08-2P screw terminal (C557665, Extended; 300 V 15 A, 24–12 AWG, D76 p1) for a panel-mount jack in the final install; pin 1 = +, pin 2 = GND. J1 is rated 5 A, just under the 5.2 A cold stall: fine for bench use, where stall is brief and current-limited. Long stall tests should use J11. No Basic/Preferred DC jack or screw terminal exists on JLC → Q1 AOD409 P-FET reverse-polarity switch (C36220, Extended; −60 V, ±20 V V_GS, D65 p1) with D2 MM1Z12 12 V gate clamp (C22379459, Preferred) + R13 10 k → +24V. D3 SMBJ24A TVS (C353373, Extended; V_C 38.9 V at 15.5 A, D66 p2), C21 100 µF 50 V electrolytic (C3151829, Extended; Roqang RVT1H101M0607, ripple rating only ~0.17 A rms, see 4.4 and #34), C22 100 nF 100 V 0805 (C28233, Basic). VBUS_SENSE = 120 k/10 k + 100 nF (38.9 V → 2.99 V). Buck: TPS5430DDAR (C9864, Basic) per D59 Fig. 7-11. Input cap C15 10 µF 50 V X5R 1206 (C13585, Basic, netdaq part; #35): D59 p13 recommends 10 µF X5R/X7R and p23 gives 4.7 µF as the minimum, and a 4.7 µF 1206 at 24 V DC bias would drop below that (general knowledge). Layout: C15 next to U1 pin 7 (VIN), small loop to pin 6 / PowerPAD (D59 p23). L1 Sunlord SWPA4020S150MT (C36407, Extended; no Basic/Preferred power inductor exists on JLC), 15 µH ±20 %, I_sat 1.35 A, I_rms 0.77 A, DCR 0.299 Ω max, 4×4×2 mm (D73 p7), KiCad footprint L_Sunlord_SWPA4020S matches the D73 p2 land pattern. Worst case ripple (D59 Eq. 6, f_SW 400 kHz min (D59 p5), L 12 µH, V_IN 38.9 V) is 0.63 A p-p, so the peak is ~0.15 A load + 0.31 A = 0.46 A, 2.9× under I_sat. A short on +3V3 (current limit 4–6 A, D59 p5) will saturate L1; hiccup mode limits that. Catch diode D1 MDD SS34 (C8678, Basic, SMA): 40 V V_RRM, 3 A, V_F 0.55 V at 3 A, T_J max 125 °C (D72 p1). D59 p16 asks for V_R > V_IN(MAX) + 0.5 V; at the full 38.9 V TVS clamp that is 39.4 V, so 40 V is just enough, and the TPS5430 has the same 40 V limit. Output cap C17/C35/C36 3× 47 µF 10 V 1206 (C96123, Basic, netdaq part), 141 µF nominal. Its DC-bias derating is not in any datasheet found (the Samsung catalog D71 and the Samsung product page have no curve), so the compensation is the D59 p22–23 design for ~100 µF effective (about 70 % of nominal): C20 100 nF, R12 510 Ω (C25123; Eq. 22 gives 516 Ω), C18 1 nF ∥ C37 470 pF = 1.47 nF (C1523 + C1537; Eq. 23 gives 1.55 nF), C19 150 pF. Feedback R10 10 k / R11 4.7 k + R27 1.2 k = 5.9 k (C25900 + C25862) → 3.29 V (Eq. 12). For 50–100 % of nominal, Fz2 lands at 2.1–3.0 × F_LC vs the 2.3–2.7 D59 recommends; check with a load step at bring-up. Bring-up test pads (not fitted by JLC, #36): TP9 +24V, TP10 +3V3, TP11 SW, TP12 GND next to the buck | No Basic/Preferred FET ≥40 V and no Basic/Preferred electrolytic exist on JLC. The TVS clamp must stay under the 40 V abs max of DRV8874 (D56 p4) and TPS5430 (D59 p4); SMBJ24A standoff is 24 V, so the PSU must stay ≤ ~25 V. Jacks rated ≥7 A in stock are mostly rated 20 V. The PSU can't absorb braking energy; firmware never returns energy to the bus (section 4.6) |
| USB | J10 HYCW498-USBC16-735B USB-C 16P (C42400252, Extended; same part as netdaq, no Basic/Preferred USB-C exists). USB FS device on PA11/PA12, D+ pull-up is internal (DPPU, D64 p2017, p2039). R23/R24 5.1 k 1 % on CC1/CC2 (C25905, Basic). D5/D6/D7 H5VUD5U ESD on DP, DM, VBUS (C20617921, Preferred; V_RWM 5 V, V_BR 6–9 V, 0.8 pF max, D68 p1–2). VBUS_USB → D8 SS14 (C2480, Basic) → U8 XC6206P332MR-G (C5446, Basic) → +3V3, C32/C33/C34 1 µF 50 V 0603 (C15849, Basic, same as netdaq; 1 µF in/out per D69 p1) | See 4.5 |
| Display | HS20HS072RX 2.0" 320×240 TFT, ST7789, on J8 (LCSC C5329582; D67) | RPM, depth, chip-load bar, torque bar, status. See 4.3 |
| Input | Rotary encoder with push button | Later option: BLDC haptic knob (MT6701 + gimbal motor) |
| Misc | Buzzer, run/stop switch (SW2), debug connectors (sheet "Debug", copied from netdaq's debug sheet, #24), UART/USB for logging | Same debug cables as netdaq. J9 HX JN1.27-2x5P ZZ H4.9 (C42372547, Extended, JLC-fitted): 2×5 1.27 mm ARM Cortex 10-pin layout, used for SWD only. Pin 2 SWDIO (R30 10 k pull-up), 4 SWCLK (R29 10 k pull-down), 10 NRST. Pins 6 (TDO) and 8 (TDI) are not connected, because PB3/PA15 are SPINDLE_B/SPINDLE_A. J12 1×6 2.54 mm header (C37208), hand-soldered (excluded from BOM and position files), FTDI TTL-232R-3V3 pinout: 1 GND, 2 CTS# NC, 3 VCC from the cable, 4 cable TX, 5 cable RX, 6 RTS# NC. U10 TLV77033PDBVR (C36996065, Extended; 1.4–5.5 V in, D8 p1; 1 µF in and ≥0.47 µF out, D8 p13) makes 3.3 V from the cable's VCC for U9 TXU0202DCUR (C5186957, Extended) VCCB. VCCA = +3V3, OE tied to +3V3 (D24 p4). With no cable, VCCB is off and the cable-side outputs are high-impedance (D24 p21). 200 Ω series R32/R33, R31 10 k pull-up on UART_RX, test pads TP1–TP7 (not fitted by JLC). Adds 3 Extended parts. J9 datasheet (datasheets/HX_JN1.27-2x5P_ZZ_H4.9.pdf) is a scan, not in the RAG; it gives no pin numbers, so the numbering follows netdaq's footprint |

Passives (issue #7): every R and C has an `LCSC Part`. Where netdaq has a Basic part with the same value, the same part and package are used (for example 100 nF 16 V 0402 C1525, 4.7 µF 25 V 0805 C1779, 1 µF 50 V 0603 C15849, 10 µF 50 V 1206 C13585, 100 nF 100 V 0805 C28233, 10 k C25744, 100 k C25741). netdaq's 10 nF (C527014) is Extended, so 10 nF uses Basic C15195 instead. Values with no Basic part are made from two Basic parts (R5+R25, R16∥R26). TPS5430 output filter and compensation (#20): see the Power row.

### 4.1 Board split

Three PCBs in one KiCad project, panelised with mouse bites:

| Board | Location | Contents |
|---|---|---|
| Main board | In front of the press, facing the user | Power input, TPS5430 buck, STM32G431, DRV8874, UI (TFT via FPC, encoder, buzzer), USB, SWD |
| Spindle sensor board | Behind the motor, on a bracket | AS5047P, decoupling, connector |
| Table sensor board | In the base, at the central post | AS5311, decoupling, connector |

- Inter-board cables are drawn as a "cable" symbol that is excluded from the board, so each board keeps its own connector footprints and nets.
- Locate the sensor boards by mounting holes, not by board edges. Breaking off the tabs leaves nubs on the edges.
- Mounting holes (#26): M3, footprint MountingHole_3.2mm_M3_ISO14580 (3.2 mm NPTH, no pad, no net), the same as netdaq. Main board H1–H4, spindle sensor board H5–H6, table sensor board H7–H8. Two holes are enough to fix position and rotation of a sensor board. Not plated and not grounded, so the metal press frame does not join GND and GND_SNS to each other or to the frame. Excluded from BOM and position files.
- To check: JLC's fee for several designs in one panel.

### 4.2 Sensor supply

- AS5047P: 3.3 V operation is supported. Tie VDD and VDD3V3 together; VDD3V3 range is 3.0–3.6 V (D60 p6, p10). ABI and UVW outputs: 1 mA max and 50 pF max capacitive load (D60 p6). Keep the cable short.
- AS5311: 3.3 V operation is supported. Bypass the LDO by tying VDD3V3 to VDD5V. VDD3V3 still needs its capacitor, and I/O levels follow VDD5V (D61 p17–18).
- Result: no 5 V rail is needed. Both sensor cables carry 3.3 V and GND.
- AS5047P TEST pin to GND (D60 p3). AS5311 Prog to VSS, NC pins left open (D61 p5).
- **AS5311 CSn must be low to enable the A/B/Index outputs** (D61 p5). Firmware keeps HEIGHT_CSn low while counting; an SSI read (pulsing CSn) interrupts the incremental outputs.
- Cable pinouts (same pin order on both ends):
  - Spindle J6 (main) ↔ J4 (sensor): custom JLCPCB flex, 12P 0.5 mm, 0.3 mm thick with stiffener; connector HC-FPC-05-20-12RLTAG (C19270548, Extended, double-sided contacts). 1 3V3, 2 GND, 3 CSn, 4 SCK (shared SPI1), 5 MISO, 6 MOSI (shared SPI1), 7 A, 8 B, 9 I, 10–12 GND
  - Height J7 (main) ↔ J5 (sensor): JST-XH 8P, B8B-XH-2 (C157973, Extended, THT), hand-crimped cable. 1 3V3, 2 GND, 3 CSn, 4 CLK, 5 DO, 6 A, 7 B, 8 I
- UI sheet: J8 display FPC (same C19270548 part); R35 100 k pull-up on LCD_CS to +3V3 (C25741, Basic, 0402), so the display stays deselected while PF0 floats in reset and boot and SPI1 talks to the spindle sensor (#32); backlight R16 10 Ω ∥ R26 47 Ω = 8.25 Ω from 3V3 (C22859 + C23182, both Basic 0603; no Basic 8.2 Ω exists) + Q2 AO3400A (C20917, Basic) low side, PWM on LCD_BL; EC11 encoder SW1 with 10 k pull-ups and 10 nF on A/B/SW. SW1 is Alps EC11E09244AQ (C470753, Extended, THT): vertical, 20 mm flat shaft (24.5 mm above the mounting surface), no threaded bushing, 18 detents / 9 pulses, push switch with 1.5 mm travel, rated 10 mA 5 V DC (D74 p1, p3 fig. 5). It is the only EC11E with a switch that LCSC stocks. With TIM3 in x4 encoder mode that is 36 counts per turn, 2 counts per detent. The KiCad footprint RotaryEncoder_Alps_EC11E-Switch_Vertical_H20mm matches fig. 5: pin rows, pitch and lug position are equal; the lug slots sit 0.1 mm further out per side (11.2 mm vs 11.0 mm centre to centre); buzzer BZ1 Jiangsu Huaneng MLT-8530 (C94599, Extended; no Basic or Preferred buzzer exists on JLC) on Q3 AO3400A with D4 1N4148W flyback. It is a magnetic, externally driven SMD buzzer, 8.5 × 8.5 × 3 mm: rated 3.6 V0-p, operating 2.5–4.5 V0-p, resonance 2700 Hz, ≥80 dB at 10 cm and ≤95 mA at 5 V0-p square wave, coil 16 ± 3 Ω (D75 p2). Because it is magnetic, D4 stays. Firmware drives BUZZER (TIM15_CH2) at 2.7 kHz, 50 % duty, and must leave the pin low when idle: held on, the coil draws DC 3.3 V / 13 Ω min ≈ 0.25 A (general knowledge, from the D75 p2 coil resistance). The EasyEDA footprint BUZ-SMD_4P-L8.5-W8.5-P8.50-BR has 2.4 mm pads at ±3.6 mm, against the D75 p4 pattern of 2.3 mm pads at ±3.55 mm; pad 1 = + lead, pad 2 = − lead, pads 3/4 = dummy pads (no net). D75 does not say whether p4 is a top view, so check the + mark on a real part before layout; SW2 run/stop switch on ENABLE_IN (issue #10): XKB Connection XKB8080-Z (C318860, Extended; no Basic or Preferred latching switch found on JLC), 8 × 8 mm THT latching push switch, DPDT, rated 0.1 A 30 VDC, 1×10⁵ cycles without load, lock stroke 2.0 mm, plunger top 16.7 mm above the board (8.15 + 5.51 + 3.00 mm), pins 2 and 5 common, 2.5 × 6.0 mm pin grid (D81 p1). Pole A: pin 2 (common) to ENABLE_IN, pin 1 to GND, pin 3 and pole B not connected. ENABLE_IN low = run, high = stop, with the 10 k pull-up R22 and 100 nF C31. D81 does not say which throw closes in the latched state, so check pins 1–2 with a meter on a real part before layout; if they close when released, move the GND wire to pin 3. Firmware runs the motor only after it sees ENABLE_IN go from stop to run after boot, so the spindle does not start at power-up with the switch left on. Mains-style power off is a panel switch in the 24 V lead to J11, not on the board. No e-stop: this is a small benchtop drill. Backlight sizing is general knowledge: (3.3 − V_f) / 8.25 Ω gives ~12–61 mA for V_f 3.2–2.8 V, always under the 80 mA abs max (D67 p5).
- On the sensor sheets, power is the sheet-local nets 3V3_SNS / GND_SNS (with PWR_FLAGs), not the global +3V3/GND. All three boards share one PCB file, so this keeps the PCB editor from drawing connections between boards. Verified in issue #14: in the netlist and in the PCB file, the sensor pads carry `/Rotary Sensor Board/…` and `/Table Height Sensor Board/…` nets, and global `GND`/`+3V3` reach main-board sheets only. No net is shared between boards.

### 4.3 STM32G431CBT6 (LQFP-48) pin map

LQFP-48 pin numbers come from the C529355 symbol. Alternate functions are from D63 Table 13 (p61–63, p66), checked against the page images. Analog functions are from D63 p52–53.

| Pin | Port | Function | AF | Net / use |
|---|---|---|---|---|
| 8 | PA0 | ADC12_IN1 | analog | VBUS_SENSE (24 V divider) |
| 9 | PA1 | ADC12_IN2, COMP1_INP, OPAMP1_VINP | analog | IPROPI, clamped to +3V3 by D9 (#33). The comparator can drive the TIM1 break internally for a fast trip |
| 10 | PA2 | USART2_TX | AF7 | Debug UART TX (J12 pin 5 via U9 level shifter) |
| 11 | PA3 | USART2_RX | AF7 | Debug UART RX (J12 pin 4 via U9 level shifter) |
| 12 | PA4 | DAC1_OUT1 | analog | DRV_VREF (current limit from software). Add a pull-down |
| 13 | PA5 | SPI1_SCK | AF5 | AS5047P CLK |
| 14 | PA6 | SPI1_MISO | AF5 | AS5047P MISO |
| 15 | PA7 | SPI1_MOSI | AF5 | AS5047P MOSI |
| 16 | PB0 | ADC1_IN15 | analog | MOT_V1 (OUT1 divider R6 120 k/R7 10 k, optional, for back-EMF/Kₑ). Sized like VBUS_SENSE: 38.9 V clamp → 2.99 V, under VDDA. PB0 is TT_a: 4.0 V abs max (D63 p69), no positive injection allowed (D63 p110). At −0.7 V on OUT1 the pin sees −54 mV, well inside the V_SS − 0.3 V minimum (D63 p69), so the D63 p124 Schottky advice for pins that inject negative current is not needed here |
| 17 | PB1 | ADC1_IN12 | analog | MOT_V2 (OUT2 divider R8 120 k/R9 10 k, optional). Same as MOT_V1; PB1 is TT_a (D63 p53) |
| 18 | PB2 | GPIO out | — | DRV_nSLEEP via R28 1 k (pull-down in DRV8874; Q4 forces low at stop) |
| 22 | PB10 | TIM2_CH3 | AF1 | SPINDLE_I (index) |
| 25 | PB11 | GPIO out | — | AS5047P CSn |
| 26 | PB12 | GPIO out | — | AS5311 CSn |
| 27 | PB13 | SPI2_SCK | AF5 | AS5311 CLK |
| 28 | PB14 | SPI2_MISO | AF5 | AS5311 DO |
| 29 | PB15 | TIM15_CH2 | AF1 | BUZZER (PWM) |
| 30 | PA8 | TIM1_CH1 | AF6 | DRV_IN1 |
| 31 | PA9 | TIM1_CH2 | AF6 | LCD_BL (backlight PWM, same period as motor PWM, own duty) |
| 32 | PA10 | TIM1_CH3 | AF6 | DRV_IN2 |
| 33 | PA11 | USB_DM | — | USB |
| 34 | PA12 | USB_DP | — | USB |
| 37 | PA13 | SWDIO | AF0 | SWD (J9 pin 2) |
| 38 | PA14 | SWCLK | AF0 | SWD (J9 pin 4) |
| 39 | PA15 | TIM2_CH1 | AF1 | SPINDLE_A |
| 40 | PB3 | TIM2_CH2 | AF1 | SPINDLE_B (gives up SWO) |
| 41 | PB4 | TIM3_CH1 | AF2 | UI encoder A |
| 42 | PB5 | TIM3_CH2 | AF2 | UI encoder B |
| 43 | PB6 | TIM4_CH1 | AF2 | HEIGHT_A |
| 44 | PB7 | TIM4_CH2 | AF2 | HEIGHT_B |
| 45 | PB8-BOOT0 | BOOT0 | — | Pull-down R1 10 k, test pad TP8 (not fitted by JLC, #25) |
| 46 | PB9 | TIM4_CH4 | AF2 | HEIGHT_I (index) |
| 2 | PC13 | TIM1_BKIN | AF2 | DRV_nFAULT (pull-up). Hardware PWM stop on fault |
| 3 | PC14 | GPIO in | — | ENABLE_IN (SW2 run/stop, low = run) |
| 4 | PC15 | GPIO in | — | UI encoder button |
| 5 | PF0 | GPIO out | — | LCD_CS |
| 6 | PF1 | GPIO out | — | LCD_DC |
| 7 | PG10-NRST | NRST | — | Reset, 100 nF, J9 pin 10 |

- Display: HS20HS072RX 2.0" 320×240 TFT, ST7789, 4-wire SPI, 12-pin 0.5 mm FPC tail (LCSC C5329582; D67). It shares SPI1 SCK/MOSI (PA5/PA7) with the AS5047P, which is only read over SPI for diagnostics; speed comes from A/B/I. Display RESET is tied to the MCU NRST. The status LED is dropped; the display shows status. IOVCC 1.65–3.3 V, VCI 2.4–3.3 V (D67 p6), so both go to +3V3. Backlight: 4 LEDs in parallel, 80 mA typ and abs max, V_f 2.8–3.2 V (D67 p5–6), driven by a low-side N-FET from PA9.
- No HSE crystal. USB runs from HSI48 trimmed by the CRS from USB SOF (D63 p26, p43). This frees PF0/PF1.
- 32-bit TIM2 counts the spindle. TIM4 counts the table, TIM3 counts the UI encoder, and TIM1 drives the bridge.
- Using PC13–PC15 only as inputs and CSn-style slow outputs is deliberate. Check their drive limits in D63 before driving any load from them.

### 4.4 DRV8874 interface

- PMODE: tie high for PWM mode (IN1/IN2). PMODE is latched when nSLEEP goes high (D56 p10).
- nSLEEP, EN/IN1 and PH/IN2 have internal pull-downs (D56 p3).
- nFAULT is open drain, 5 mA max; pull up to 3.3 V (D56 p3, p4).
- VREF range is 0–3.6 V (D56 p4). I_TRIP = V_VREF / (R_IPROPI × A_IPROPI) (D56 p13). A_IPROPI = 450 µA/A typ, error ±5.5 % at 2–4 A (D56 p6 table). TI's worked examples use 455 (p13, p21).
- R_IPROPI = 1.33 kΩ 1 %, made as R5 1 kΩ + R25 330 Ω in series (C11702 + C25104, both Basic 1 %; no Basic 1.33 kΩ exists). Full-scale DAC (3.3 V) gives 5.5 A typ, or 5.85 A at −5.5 % gain. Either way no DAC value can set the limit above the 6 A peak rating. At 5.2 A stall, IPROPI ≈ 3.1 V, inside the ADC range.
- At reset the DAC is off and R4 (100 k) holds VREF at 0 V. The bridge then cannot drive current until firmware sets the limit, so it fails safe.
- IMODE = GND through R2 0 Ω (C17168, Basic): fixed off-time chopping, automatic retry after overcurrent, and nFAULT reports overcurrent only (D56 Table 6, p13). Fixed off-time supports 100 % duty (D56 p14). Not 20 kΩ (cycle-by-cycle): there nFAULT also goes low on every chop (D56 p14), so TIM1_BKIN would stop all TIM1 outputs, LCD_BL included, at each current limit, and 100 % duty is not possible (issue #30). R2 stays as a 0 Ω part so 20 kΩ or 62 kΩ can be fitted later.
- C14 (10 nF on IPROPI) is DNP. D56 p13 suggests it only if transients trigger the chopping early.
- IPROPI clamp (issue #33): D9 RB751V-40 (C7502691, Preferred, SOD-323), anode on IPROPI, cathode on +3V3. PA1 is TT_a (D63 p52), max 4.0 V (D63 p69). Without the clamp, IPROPI goes over 4 V above about 6.7 A, and OCP trips at 6 A min / 10 A typ only after t_OCP 3 µs (D56 p6), on every retry. IPROPI itself is rated to 5.75 V (D56 p4). IPROPI is a current output, so the diode alone limits the voltage. There is no series resistor, so PA1 keeps full resolution and the same source impedance. Vf ≤ 0.37 V at 1 mA, about 0.4 V at 10 mA (D85 p2), so PA1 stays at or below about 3.7 V. In normal running VREF comes from the DAC, so VREF ≤ +3V3, IPROPI ≤ +3V3, and the diode is off. Leakage is < 1 nA at 25 °C and about 10 nA at 100 °C (D85 p2), ≤ 13 µV across 1.33 kΩ. Capacitance is about 15 pF at 0 V (D85 p2), which adds about 20 ns. Layout: place D9 next to PA1.
- nSLEEP has an internal 100 kΩ pull-down (D56 p5), so it gets no external pull-down.
- Hardware stop (issue #11), on the MCU sheet: Q4 AO3400A (C20917, Basic) pulls DRV_nSLEEP to GND when ENABLE_IN is high (SW2 at stop), whatever the MCU does. Gate on ENABLE_IN, drain on DRV_nSLEEP, source on GND. PB2 drives DRV_nSLEEP through R28 1 kΩ 0402 (C11702, Basic), so PB2 high against Q4 on draws 3.3 mA instead of a short. Numbers: Q4 V_GS(th) ≤ 1.45 V and R_DS(on) ≤ 48 mΩ at V_GS 2.5 V (D82 p2), so 3.3 V on the gate turns it fully on; nSLEEP is then ≈ 0 V, under V_IL max 0.8 V (D56 p5). With Q4 off and PB2 high, nSLEEP sits at 3.3 V minus at most 75 µA × 1 kΩ = 75 mV, over V_IH min 1.5 V (D56 p5). The driver enters sleep within t_SLEEP 1 ms (D56 p5), and the motor coasts. R22/C31 (10 k, 100 nF) slow the gate by about 1 ms on the way to stop. Firmware still reads ENABLE_IN on PC14 and must also drop PB2.
- VM ripple (issue #34): C11, C42 and C43 are 10 µF 50 V X5R 1206 (C13585, Basic, netdaq part) on +24V at U5 VM, next to C10 100 nF. D56 p10 asks for 0.1 µF low-ESR ceramic plus "bulk capacitance, VM-rated", and p31 asks for low-ESR X5R/X7R placed close to VM. With drive/coast PWM the input ripple is about I·√(D(1−D)), 2.5 A rms at 5 A and 50 % duty (general knowledge). C21 (Roqang RVT1H101M0607, 100 µF 50 V, 6.3 × 7.7 mm) is rated 114 or 130 mA rms at 120 Hz, 105 °C (the 50 V 100 µF 6.3 × 7.7 row has both, D86 p2), times 1.40 at ≥ 10 kHz (D86 p1), so about 0.16–0.18 A. That is far under the stall ripple. Not verified: that a stall is short, or that C21 survives a short overload (D86 says nothing about overload or how fast it heats). D86 gives no ESR at 20–40 kHz, so how the ripple splits between the ceramics and C21 is not known: the ceramics may take most of it, or C21 may run hot. Once #16 gives the running current, and at bring-up during a stall, measure C21's ripple current or temperature rise. Replace C21 with a higher-ripple part if it is over. Layout: C11, C42, C43 next to U5 pin 11 (VM) with a short loop to pin 9 (PGND).
- Continuous current / thermal limit for U5 (issue #40). ITRIP (VREF) only caps the peak; current chopping can hold a stall at ITRIP for as long as firmware lets it, so the heat limit is a firmware job. Numbers:
  - Losses: P = I² × (R_DS(on)_HS + R_DS(on)_LS) + P_SW + P_VM (D56 p22 Eq. 11–19). R_DS(on) is 120 mΩ max per FET at 25 °C (D56 p6) and about 1.5× that at 150 °C (Fig. 6/7, D56 p8), so the hot worst case is 0.36 Ω for the path. P_SW = 0.5 × 24 V × I × (150 + 150 ns) × f_PWM = 0.072 W/A at 20 kHz, double at 40 kHz (t_RISE/t_FALL D56 p6). P_VM ≈ 0.1 W (D56 p22 Eq. 10).
  - Target: T_J ≤ 150 °C, 10 °C under the 160 °C min thermal shutdown (D56 p6). Assumed T_A 40 °C inside the case (not measured).
  - R_θJA depends on the board, not the 36 °C/W in the thermal table (D56 p5, p22). TI's simulation, 1 oz copper (D56 p24 Fig. 21 and 23): 2-layer 63 °C/W at 4 cm² top copper, 44 °C/W at 16 cm²; 4-layer 31.5 °C/W at 4 cm², 30.4 °C/W at 16 cm².
  - Continuous limit at 20 kHz: 2-layer about 2.0 A (4 cm²) to 2.5 A (16 cm²); 4-layer about 3.0 A. The stackup is not decided: the PCB file is 2-layer now, and the choice waits for the running current from #16. If the drilling current is above the 2-layer limit, go 4-layer; if it is above about 3 A, see the DRV8873 fallback in the motor driver row of the section 4 table.
  - TI's model has thermal vias on a 1.2 mm grid under the whole pad (D56 p22–23); our footprint has 4 (#15). So the real limit is probably lower than these numbers (not quantified). Bring-up: run a known current for a few minutes and measure the U5 case temperature, then correct R_θJA.
  - Stall: 5.2 A gives about 10 W. Z_θJA reaches about 11 °C/W at about 0.3 s on every stackup (D56 p25 Fig. 25), so from 40 °C the junction reaches 150 °C in about 0.3 s. Thermal shutdown (D56 p17) is the backup, not the plan.
  - **Firmware rule: U5 I²t limit.** Estimate U5 power from the IPROPI current with the formula above, and T_J with a thermal model fitted to Z_θJA (D56 p25). Fold the current back to the continuous limit for the chosen stackup before T_J reaches 150 °C. A locked rotor (current at ITRIP, speed near 0) must cut the drive within 0.2 s. This is separate from the motor I²t in section 5; the lower of the two wins.
- MOTOR_A and MOTOR_B leave the sheet as hierarchical labels to J3 on the main board sheet (issue #22). J3 is the same part as J11: Xinlaiya XY128V-A-5.08-2P screw terminal (C557665, Extended; 300 V 15 A, 24–12 AWG, D76 p1), so it adds no feeder fee. Pin 1 = MOTOR_A, pin 2 = MOTOR_B. 15 A is about 3× the 5.2 A cold stall.

### 4.5 USB power path (issue #1)

- USB powers +3V3 through U8 so the MCU can be flashed and logged without the 24 V PSU.
- D8 sits in front of U8 so +3V3 cannot back-drive the host through U8 when only the PSU is on. U8 and the TPS5430 both only source current; whichever output is higher carries the load.
- USB only: +3V3 also back-feeds the TPS5430 output, and (general knowledge, not in D59) through the high-side FET body diode into +24V, which then sits near 2.7 V. The loads there are small (VBUS_SENSE divider, sleeping DRV8874, TPS5430 below UVLO). Accepted as is; reg_3v3 is unchanged.
- VBUS_SENSE is then about 2.7 V × 10/130 ≈ 0.2 V, so firmware can tell "no PSU".
- U8 limits: I_OUT ≥ 200 mA, dropout 250 mV typ / 680 mV max at 100 mA (D69 p5), 250 mW in SOT-23 (D69 p3), V_IN max 6.0 V (D69 p4).
- USB-only load budget: MCU 24.5 mA typ at 170 MHz, peripherals off (D63 p81), AS5047P 15 mA max (D60 p6), AS5311 21 mA max (D61 p7), backlight up to 61 mA (D67 p5, see 4.2), display logic not specified (D67 p6). About 150 mA worst case → ~0.24 W in U8 at 5 V in, right at the 250 mW limit.
- **Firmware rule: when VBUS_SENSE shows no PSU, cap the backlight.** Without the backlight the load is about 90 mA, ~0.15 W in U8.
- **Firmware rule: when VBUS_SENSE shows no PSU, keep BUZZER (PB15) low; the buzzer is muted** (issue #39). BZ1 is rated max 95 mA at 2700 Hz, 50 % duty, 5 V0-p (D75 p2). At 3.3 V that scales to roughly 63 mA (general knowledge, linear estimate). On top of the 150 mA budget that is over U8's 200 mA. The motor cannot run on USB only, so the buzzer has nothing to signal there. Not chosen: a series resistor (would also make it quieter on PSU power) and local bulk at BZ1 (+3V3 already has C17/C35/C36, 141 µF nominal).
- Shell pads of the EasyEDA C42400252 footprint were renumbered to pad 1 (SHELL) so all four connect to GND.

### 4.6 Braking energy (issue #3)

Decision: handle it in firmware. No brake resistor, no extra bulk capacitance.

- The PSU cannot sink current, so any energy the motor returns raises +24V. The bus must stay under the 40 V abs max of the DRV8874 (D56 p4) and the TPS5430 (D59 p4).
- Brake (IN1 = IN2 = 1) turns on both low-side FETs (D56 p11, Table 4). The motor is shorted through them, and the energy is burned in the winding and the FETs. Nothing flows to +24V.
- Current chopping also uses this low-side brake state (D56 p13), so it does not pump the bus either.
- Coast (IN1 = IN2 = 0) is Hi-Z (D56 p11). The back-EMF stays below the bus voltage, so the body diodes do not conduct to +24V (general knowledge).
- Brake current is at most back-EMF / R ≈ 24 V / 4.66 Ω ≈ 5.2 A at full speed, the same as stall, and below the 6 A peak rating (D56 p4).
- The bus is pumped only when the bridge is driven with a duty below the back-EMF ratio while the low side recirculates (drive/brake PWM at low duty). Then the motor current reverses and flows into +24V on each on-time (general knowledge).
- **Firmware rule: use drive/coast PWM. Never command negative motor current. Slow down only with coast or full brake.** The current loop in section 5 clamps its reference at ≥ 0.
- **Firmware rule: if VBUS_SENSE reads above 30 V (2.31 V at the ADC), set the bridge to coast and log a fault.**
- Cost: no controlled regenerative deceleration. Stops are either coast (slow) or full brake (fast). That is enough for bit changes (section 6, Active brake).

## 5. Control architecture

1. **Current loop** (PWM rate, 20–40 kHz): PI with back-EMF feedforward (Kₑ·ω̂). This loop sets torque.
2. **Observer** (about 10 kHz): disturbance observer that fuses the ABI speed with current and voltage, and outputs ω̂ and the load-torque estimate T̂_load.
3. **Load feedforward**: T̂_load / Kₜ added to the current reference. This is what reacts fastest when the bit bites.
4. **Speed loop** (1–2 kHz): PI that trims the remaining error and generates the setpoint ramp.
5. **Limits**: current/torque limit per bit preset, plus two thermal estimates (I²t): one for the motor winding and one for U5 DRV8874 (section 4.4, issue #40). The lower limit wins.

Parameter ID at startup: lock rotor briefly to get R and L, spin up to get Kₑ, coast down to get J and friction. Track R with temperature in operation.

## 6. Bit protection and process features

| Feature | Inputs | Action |
|---|---|---|
| Chip-load monitor | Feed rate (d(height)/dt) ÷ RPM | Live bar; beep and on-screen warning when over the preset limit (µm/rev) |
| Torque watchdog | Current, dI/dt | Hard torque limit; alarm on rapid rise ("retract / peck") |
| Auto-zero on contact | Current step + height | Sets depth zero at the board surface |
| Breakthrough detect | Current drop + height ≈ thickness | "Through" indicator |
| Bit presets | Encoder menu | RPM, chip-load limit, torque limit per bit diameter |
| Hole counter / wear log | Mean cutting current per hole | Flags a dull bit when the trend rises |
| Active brake | Driver | Fast stop for bit changes |

## 7. Firmware structure

- `hal/`: PWM, ADC, encoder timer, SPI1 (AS5047P and ST7789 display, shared bus), height sensor
- `control/`: current PI, observer, speed PI, limits, parameter ID
- `process/`: chip-load, torque watchdog, contact/breakthrough, presets, counters
- `ui/`: display screens, encoder menu, alarms
- `log/`: UART/USB streaming of ω, i, T̂, height for tuning

Run control in ISRs triggered by the PWM timer; run UI and process logic in the main loop or a low-priority task.

### 7.1 UCPD dead-battery pull-downs (issue #19)

- After reset the STM32G431 can put Rd = 5.1 kΩ to GND on PB6 and PB4. A high level on PA9 turns it on for PB6, and a high level on PA10 turns it on for PB4 (D63 p60). The feature is on after reset; `PWR_CR3.UCPD1_DBDIS` (bit 14) = 1 turns it off, and RM0440 recommends turning it off in all cases (D64 p255).
- On this board PA9 = LCD_BL PWM, PA10 = DRV_IN2 PWM, PB6 = HEIGHT_A, PB4 = ENC_A. Without the fix, each high PWM period puts 5.1 kΩ on HEIGHT_A and ENC_A. ENC_A (10 k pull-up R18) would then read ≈ 3.3 V × 5.1 / 15.1 ≈ 1.1 V.
- **Firmware rule: set `PWR_CR3.UCPD1_DBDIS = 1` first thing at startup, before PA9 or PA10 is set up as an output or AF.** UCPD1 is not used, so nothing needs to be set up before it.
- Before firmware runs, PA9 and PA10 are in analog mode (GPIOA_MODER reset value 0xABFF FFFF, D64 p355), and each net is held low: LCD_BL by R17 100 kΩ to GND, DRV_IN2 by the DRV8874 internal 100 kΩ pull-down on PH/IN2 (D56 p3, p18 fig. 16). So the pull-downs stay off during reset and boot. No hardware change.
- Not checked: the ST system bootloader (BOOT0 high, or empty flash) may use PA9/PA10 as USART1 (general knowledge, not in D63/D64 as searched). That only matters while the bootloader runs; the motor driver stays asleep then, because nSLEEP is pulled low.

## 8. Phases

1. **Characterise the motor**: measure R, L, Kₑ, J and the no-load current on the bench, and log current while drilling real holes.
2. **Sensor bring-up**: mount the magnet, put an AS5047P on a breakout, and check speed ripple at 20k rpm (magnet centring).
3. **Board rev A**: MCU, driver, current sense, AS5047P connector, TFT display, encoder, debug and logging.
4. **Control loops**: current loop, then speed PI, then observer and feedforward. Tune with logged load-step tests.
5. **Height sensor**: fit it, calibrate it, add the depth readout.
6. **Process features**: chip-load and torque watchdog first, then auto-zero, breakthrough and presets.
7. **Refinement**: thermal tracking, wear log, optional haptic knob.

## 9. Risks

- Magnet runout or imbalance shows up as once-per-revolution speed ripple (333 Hz at 20k rpm). Needs a press-fit cap and filtering.
- Brush and commutator noise on the current signal. Filter it and sample synchronously with the PWM.
- Lever linkage is nonlinear with pin play, which is why the table is measured directly rather than at the lever.
- Printed head arm flexes under feed force. Small for micro bits, but the table sensor won't see it.
- Regenerative braking pumps the 24 V bus, because a barrel-jack PSU can't sink current. Handled by firmware rules in section 4.6.
- Hot-plugging the barrel jack causes inrush and a voltage spike. Decision (#37): no inrush limiter; the risk is accepted. Q1's body diode (drain on VIN_RAW, source on +24V) charges +24V at plug-in, so an RC on the Q1 gate cannot slow it, and real soft start would need a second FET or a hot-swap IC. +24V has only about 140 µF (C21 100 µF + C11, C15, C42, C43 4× 10 µF), which is a small charge at 24 V. The spike comes from cable inductance ringing with the low-ESR ceramics; C21's ESR damps it and D3 SMBJ24A clamps it to 38.9 V at 15.5 A (D66 p2), under the 40 V abs max of DRV8874 (D56 p4) and TPS5430 (D59 p4). The damping and the inrush peak are general knowledge, not measured. A series NTC was rejected: it drops power at the 5 A stall and does not limit on a quick re-plug while hot. Bring-up: scope +24V at TP9 while hot-plugging J1 with the bench PSU and its cable, and check that the peak stays under 40 V. If it does not, revisit this.
- No input fuse between J1/J11 and +24V. Decision (#38): the PSU's own short-circuit protection is relied on. If D3, Q1 or U5 fails short, the PSU's limit current flows through J1 (5 A, D78 p1) and the board copper until the PSU shuts down. A fuse was rejected: it must carry the 5.2 A cold stall, so it would be about 10 A, and that does not blow at the 6–9 A a 150 W brick gives into a short (general knowledge), so it would not protect J1. JLC has no Basic or Preferred fuse (jlcsearch, 2026-11-22), so it would also add a feeder fee. A PTC was rejected: the only ≥ 5 A SMD part found (2920L500/30GR, C19078763) holds 5 A, below the stall, and is rated 30 V, below the 38.9 V TVS clamp. So the PSU must have short-circuit/overcurrent protection (hiccup or foldback), see the supply item in section 2. A lab supply must have its current limit set to 6 A or less.
- AS5047P propagation delay at high speed. Use ABI with DAEC enabled.
