# Hardware: units and bridge wiring

This page covers the ventilation unit variants and the wiring of the ESP32 bridge. For the register map see the [README](../README.md).

## Unit variants

| Model | Airflow | Status in this repo |
|---|---|---|
| Vitovent 300-W, Typ H32E C325 (enthalpy) | 325 m³/h | Tested (register scan and USR-TCP232 gateway, see README) |
| Vitovent 300-W, Typ H32S C400 (R) | 400 m³/h | Listed. Register map not verified on this variant. Possibly an older generation (2013), see below |

The register map in the README was derived from the H32E C325. Check the other variants before relying on it.

### Check the unit generation first

Older units may use a different control board or a different register layout. A unit from around 2013 likely belongs to an older generation, and the register map may not apply to it.

Before any write:
1. Open the unit (or have a qualified person do it) and identify the control board. The tested generation has a **UWA2-B** board with connector **X15**.
2. Photograph the board, its label and the connectors, and compare them with the tested unit.
3. Do not write register values unless the board and the register behaviour have been verified on that unit.
4. Only one Modbus master may be active on the bus at a time.

## Verschaltung (Übersicht)

Die Komponenten werden in dieser Reihenfolge verbunden:

```
AtomS3 Lite           Eigenbau-Kabel         Tail485                 Vitovent 300-W
PORT.A                                       TTL-Seite   RS485-Seite  X15
 Pin 1 GPIO1 (RX) ----------------------------- RX  (RO) ... A  ------------- Pin 2 (A+)
 Pin 2 GPIO2 (TX) ----------------------------- TX  (DI) ... B  ------------- Pin 1 (B-)
 Pin 3 5 V       ------------------------------ 5 V
 Pin 4 GND       ------------------------------ GND ------------------------- Pin 3 (GND)
```

- Der AtomS3 Lite steckt in PORT.A. Die Belegung ist unten (Tabelle) und wird in `vitovent-300w.yaml` über `modbus_rx_pin` und `modbus_tx_pin` gesetzt.
- Das Kabel verbindet PORT.A (2,0 mm) mit dem Tail485 (VH-3.96). Die Farben für die Adern nach dem M5-Standard: G1 = weiß, G2 = gelb, 5 V = rot, GND = schwarz.
- Der Tail485 bekommt die RS485-Leitungen A und B auf den VH-3.96-Stecker. A geht an Pin 2 von X15, B an Pin 1.
- Die Signal-GND muss auf X15 Pin 3 gelegt werden. Ohne sie antwortet der Bus laut README nicht.

| Verbindung | Von | Nach |
|---|---|---|
| RX (ESP → Empfang) | AtomS3 PORT.A Pin 1 (GPIO1) | Tail485 RX (RO) |
| TX (ESP → Senden) | AtomS3 PORT.A Pin 2 (GPIO2) | Tail485 TX (DI) |
| Spannung | AtomS3 PORT.A Pin 3 (5 V) | Tail485 5 V |
| Masse | AtomS3 PORT.A Pin 4 (GND) | Tail485 GND |
| Bus A | Tail485 A | X15 Pin 2 (A+) |
| Bus B | Tail485 B | X15 Pin 1 (B−) |
| Bus GND | Tail485 GND | X15 Pin 3 (GND) |

**Status:** Die Verkabelung über PORT.A mit der Tail485-Belegung (GPIO1 = RX, GPIO2 = TX) läuft im Funktionstest stabil. Die Richtungsumschaltung arbeitet ohne Steuerpin. Die RTS-Leitung wird nicht genutzt.

Offen: ein formaler Loopback- oder Oszilloskop-Test zum Echo und zu Kollisionen (#3).

## Bus connection (unit side, X15)

The X15 pinout and the serial parameters are in the [README](../README.md#2-physical-connection) (section 2). They are not repeated here.

## Bridge wiring: AtomS3 Lite + Tail485

The ESPHome config (`vitovent-300w.yaml`) uses PORT.A of the AtomS3 Lite.

**ESP side (AtomS3 Lite, PORT.A)**

| PORT.A pin | Signal | Tail485 side |
|---|---|---|
| 1 | GPIO1 (RX) | RO / Tail485 "RX" |
| 2 | GPIO2 (TX) | DI / Tail485 "TX" |
| 3 | 5 V | 5 V |
| 4 | GND | GND |

PORT.A uses a 2.0 mm connector (GH2.0 on the schematic, HY2.0-4P in the M5 docs). The Tail485 uses VH-3.96 4P. No ready-made adapter was available, so the cable has to be built by hand.

**Tail485 to bus**

The Tail485 contains an SP485-family transceiver. Its pins (Sipex SP481E/SP485E datasheet, pin-compatible family):

| Chip pin | Name | Connect to |
|---|---|---|
| 1 | RO | ESP RX (GPIO1) |
| 4 | DI | ESP TX (GPIO2) |
| 6 | A | X15 pin 2 (A+) |
| 7 | B | X15 pin 1 (B−) |
| 5 | GND | GND |
| 8 | VCC | 5 V |
| 2, 3 | /RE, DE | Direction control on the board (auto, to verify) |

Status: wired and working in the functional test (stable operation). The RX/TX assignment above is confirmed by operation. The direction switching works without a control pin.

Still open: a formal loopback or scope test for echo and collisions (#3).
- Wire polarity before connecting to the unit: A to pin 2, B to pin 1.
