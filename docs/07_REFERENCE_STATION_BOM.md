# Raspberry Pi 4 Reference Station BOM

## Goal

Build one repeatable Embedded Linux validation station capable of:

```text
register image
  -> acquire station lease
  -> provision Raspberry Pi 4 boot media
  -> capture UART
  -> power/reset DUT
  -> wait for SSH
  -> run Linux Core tests
  -> collect evidence
  -> restore known-good state
```

This station is the M1 reference platform. It is not a customer production SLA
station and does not initially include power measurement, cellular, CAN, or
environmental chambers.

## Roles

### Control and build plane

Existing Xeon server:

- Jenkins and build agent;
- artifact storage;
- Prometheus and Grafana;
- optional internal Yocto reference image builds;
- report generation.

The Xeon must not directly access DUT hardware.

### Station controller

Dedicated mini PC or existing always-on x86/ARM controller:

- Jenkins station agent or future station worker;
- Docker runtime;
- USB access to UART, relay, and media switch;
- wired Ethernet;
- enough local storage for current artifacts and evidence;
- separate from the DUT.

Do not use the Raspberry Pi 4 DUT as its own controller.

### DUT

Raspberry Pi 4 Model B:

- 4 GB or 8 GB RAM;
- board revision recorded in `HardwareConfig`;
- official or equivalent stable PSU;
- heat sink/fan for repeatable long runs;
- UART exposed through fixture;
- wired Ethernet for M1;
- boot order configured for the chosen media.

## Required BOM

### Compute and DUT

- existing Xeon build/CI server;
- one dedicated station controller;
- one Raspberry Pi 4 DUT;
- one spare Raspberry Pi 4 when budget allows;
- one high-quality Pi power supply per board.

### Provisioning

Preferred automated option:

- SDWire or equivalent SD-card multiplexer compatible with Pi 4 and controller;
- one high-endurance microSD card for active testing;
- one separate known-good recovery card;
- USB connection from multiplexer to station controller.

Fallback during initial bring-up:

- two high-endurance microSD cards;
- USB 3 card reader with stable serial identity;
- documented manual transfer;
- automation begins only after target media allow-list is proven.

Manual transfer is acceptable for first local bring-up but not for the final
2–5 minute automated customer demo.

Alternative USB boot storage may be evaluated, but a shared disk must never be
simultaneously writable by the controller and DUT. A normal USB hub or relay
does not provide safe host switching.

### Console

- 3.3 V USB-to-UART adapter with a stable USB serial number;
- RX/TX/GND wiring;
- keyed or labeled connector;
- no 5 V UART signaling;
- optional second UART adapter as spare.

Record:

- USB VID/PID and serial;
- `/dev/serial/by-id` mapping;
- baud rate;
- fixture pinout.

### Power and reset

- USB-controlled relay or managed power switch rated for the Pi PSU;
- normally safe power-off behavior;
- station-controllable power state;
- physical manual override;
- spare relay channel if available.

Do not switch high voltage with an exposed hobby relay. Prefer a certified
enclosed smart PDU/relay or switch only the low-voltage DC side with suitable
ratings and protection.

### Network

- wired Gigabit Ethernet switch;
- dedicated station/DUT network segment or VLAN;
- DHCP reservation or controlled discovery;
- firewall policy for DUT outbound access;
- backup network path for station controller if practical.

Customer images must not share an unrestricted trusted home or office LAN.

### Reliability and evidence

- UPS covering controller, network switch, relay control, and CI services where
  practical;
- fixed webcam showing only DUT/fixture;
- temperature sensor optional for station context;
- labeled cables and fixed mounting;
- spare microSD, USB-UART cable, Ethernet cable, and PSU;
- non-personal camera background.

## Recommended topology

```text
Xeon CI/build server
       |
       | trusted control network
       v
Station controller
  |-- USB -> SD multiplexer -> Pi 4 boot media
  |-- USB -> UART adapter -> Pi 4 console
  |-- USB/network -> power relay -> Pi 4 PSU
  |-- USB/network -> webcam
  |
  +-- isolated test network -> Pi 4 Ethernet

Raspberry Pi 4 DUT
```

## Stable device mapping

Never provision using the first matching `/dev/sdX`.

Station configuration must map:

- media writer by USB serial or approved topology;
- UART by `/dev/serial/by-id`;
- relay by controller identity/channel;
- webcam by stable device identity;
- DUT network by controlled lease/MAC mapping where policy permits.

Before every destructive write:

1. active station lease and fencing token are valid;
2. device identity matches allow-list;
3. device capacity is within expected bounds;
4. device is not the controller root or mounted system disk;
5. artifact role and checksum match the run snapshot;
6. DUT power state is safe for media switching.

## Provisioning flow

For Yocto `.wic.bz2` plus `.wic.bmap`:

1. verify artifact size and SHA-256;
2. switch media ownership to controller;
3. confirm allow-listed block device;
4. write using `bmaptool` with decompression support or a controlled equivalent;
5. flush buffers;
6. verify written content according to policy;
7. unmount/eject cleanly;
8. switch media ownership to DUT;
9. start UART capture;
10. power on DUT.

Provisioning evidence records tool versions, device identity, bytes written,
duration, checksum, and exit status.

## Known-good recovery

Maintain:

- immutable known-good image artifact;
- known-good run ID;
- separate recovery media where practical;
- maximum recovery attempts;
- quarantine rule after repeated failures.

Recovery must be tested before public demo. A failed regression image must not
leave the station requiring manual intervention for the next scheduled demo.

## Deferred equipment

Buy only after a real test pack or customer requires it:

- Joulescope or precision power analyzer;
- USB-CAN/CAN-FD;
- BLE sniffers;
- cellular modems/SIM bank;
- RF enclosure;
- programmable PSU;
- thermal chamber;
- customer-specific fixtures.

## Station reactivation checklist after relocation

- inspect power, wiring, grounding, and physical mounting;
- confirm UPS and safe shutdown;
- confirm controller and Jenkins agent connectivity;
- confirm Docker runtime and device permissions;
- inventory USB devices by stable identity;
- test relay power cycle without DUT media corruption;
- validate UART capture;
- execute ESP32 PASS/FAIL flow at least 10 consecutive times;
- archive one golden ESP32 evidence set;
- only then connect/provision the Pi 4 reference DUT.

## Procurement gate

Before buying an SD multiplexer, confirm:

- Pi 4 electrical/mechanical compatibility;
- Linux host tooling support;
- stable device identity;
- ability to prevent simultaneous host/DUT access;
- replacement availability;
- shipping lead time to the lab location.
