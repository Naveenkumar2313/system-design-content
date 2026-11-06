# CMP-001 — Visual Component Specification

## Canvas identity

| Field | Value |
|---|---|
| Display name | Load Balancer |
| Component ID | CMP-001 |
| Icon concept | Network / traffic distribution |
| Node type | Infrastructure |
| Default label | Load Balancer |
| Default status | Healthy |

## Ports

| Port | Direction | Protocol / semantic type | Description |
|---|---|---|---|
| client_in | Input | HTTP / HTTPS / TCP | Incoming client traffic |
| backend_out | Output | Forwarded request / connection | Traffic forwarded to a target |
| telemetry_out | Output | Metrics / logs | Operational telemetry |

## Properties panel

- Routing algorithm.
- Listener protocol and port.
- Health-check configuration.
- Backend target list.
- Backend weights.
- Connection/request timeout.
- TLS configuration (where applicable).
- Session persistence (where supported).

## Connection rules

- Client or gateway may connect to the frontend input.
- Backend services may connect to backend outputs.
- A load balancer should have at least one backend target to run a useful simulation.
- Reject incompatible protocol connections with a clear explanation.
- Support one-to-many backend connections.

## Visual states

- Healthy: accepting traffic.
- Degraded: some backends unavailable.
- Overloaded: offered load exceeds modeled capacity.
- Unavailable: no eligible backend targets or load balancer unavailable.

## Runtime behaviour

- Animate requests from frontend to selected backend.
- Display per-backend request counters.
- Update target health state when failure injection changes.
- Show routing policy in node details.
- Provide a link to the component learning documentation.
