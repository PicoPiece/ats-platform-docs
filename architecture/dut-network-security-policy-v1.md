# DUT Network Security Policy v1

## Purpose

A customer image is untrusted code running on real hardware. It may be
misconfigured, compromised, or intentionally hostile.

Network isolation is therefore an M2 entry requirement, not an optional
operational improvement.

## Trust zones

```text
ControlPlane
  Jenkins, registry, databases, secrets, admin access

StationManagement
  station controller management interface and hardware-control services

DutVlan
  customer/reference DUT network

ApprovedServices
  explicit DNS/NTP/update/test endpoints or controlled proxies

Internet
  denied by default
```

Required topology:

```text
DutVlan
   |
Firewall
   +--> ApprovedServices  # explicit profile rules only
   X--> ControlPlane
   X--> HomeOrOfficeLan
   X--> StationManagement
   X--> Internet          # default

StationRunner
   +--> DUT required ports only
```

Physical implementation may use VLANs, dedicated interfaces, or a dedicated
firewall/router. Logical separation without enforceable firewall policy is not
sufficient.

## Default-deny rules

By default:

- DUT to control plane: deny;
- DUT to Jenkins, registry, databases, and secret stores: deny;
- DUT to home/office LAN: deny;
- DUT to station management interface: deny;
- DUT to other DUTs/stations/tenants: deny;
- DUT to Internet: deny;
- inbound Internet to DUT: deny;
- station runner to DUT: allow only declared protocol/ports;
- control plane to DUT: no direct path; use the station runner;
- DNS and NTP: deny unless supplied by an approved service profile.

Layer-2 controls must prevent access to management VLANs and unintended peers.
IPv4 and IPv6 policy must be equivalent; disabling only one protocol is not
isolation.

## NetworkAccessProfile

Every run resolves an immutable network profile, for example:

```yaml
profile_id: dut-offline-v1
egress:
  default: deny
  rules: []
ingress:
  from_station:
    - protocol: tcp
      port: 22
dns: disabled
ntp: disabled
east_west: deny
bandwidth_limit_mbps: 100
connection_limit: 256
```

An Internet-enabled profile must explicitly define:

- destination domains/IP ranges or controlled proxy;
- ports/protocols;
- DNS resolver;
- bandwidth and connection limits;
- logging policy;
- business/test justification;
- expiry/review date.

`allow any Internet` is not an acceptable default customer request. Exceptions
require review and are scoped to the run/project.

## Station controller design

The station controller should have separate logical or physical interfaces for:

- management/control communication;
- DUT test network.

The DUT-facing service surface is minimized:

- SSH client connections from station to DUT;
- declared API/MQTT/test endpoints;
- optional controlled DHCP/DNS/NTP;
- no Docker socket, Jenkins agent port, secret store, or host admin service.

Hardware control APIs bind only to the management interface or local socket.

## Threats addressed

- network scanning;
- malware callback/exfiltration;
- crypto-mining traffic;
- attacks against Jenkins/control services;
- attacks against home/office devices;
- cross-customer/DUT traffic;
- accidental DHCP/DNS conflicts;
- IPv6 bypass;
- excessive bandwidth or connection exhaustion.

Isolation reduces risk but does not make customer code trusted.

## Observability and privacy

Record:

- network profile ID;
- firewall policy revision;
- blocked/allowed connection metadata according to retention policy;
- DHCP assignment;
- bandwidth/connection-limit events;
- detected policy violations.

Do not capture customer payload contents by default. Packet capture requires
declared test scope, retention, and customer approval.

## Violation handling

On material policy violation:

1. record timestamp and rule;
2. stop or isolate the DUT network;
3. mark the run `completed` / `ERROR` unless a declared security test owns the
   assertion;
4. preserve permitted evidence;
5. power off or recover the DUT;
6. quarantine the station/release when required;
7. require review before rerun.

If the declared test intentionally validates forbidden traffic behavior, the
test may produce `FAIL`; infrastructure enforcement failure remains `ERROR`.

## Pre-M2 acceptance tests

- DUT cannot reach Jenkins/control-plane addresses;
- DUT cannot reach home/office LAN;
- DUT cannot reach another DUT VLAN;
- DUT cannot reach Internet under offline profile;
- station can reach only declared DUT ports;
- IPv6 cannot bypass policy;
- explicit DNS/NTP profile works without broader access;
- explicit restricted-Internet profile reaches only approved destinations;
- connection and bandwidth limits are enforced;
- firewall policy survives controller/firewall restart;
- policy ID and revision appear in run snapshot/report.
