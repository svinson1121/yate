# MGCP SS7 Bearer Bridging

## Purpose

This note documents the MGCP bearer-bridging model for SS7/ISUP calls when multiple signalling networks share one Yate MGW.

Typical topology:

`SIP/ISUP network A -> SS7/ISUP -> Yate MGW <- SS7/ISUP <- SIP/ISUP network B`

and likewise for `A/B/C/D/...` sharing the same MGW.

The goal is:

- keep ISUP for call control
- keep MGCP for bearer control
- let the MGW bridge the two bearer legs dynamically
- avoid hardwired endpoint-pair rules such as `acgw1xx <-> acgw2xx`


## Problem

The old ISUP example often routes MGW-side calls to `dumb/`.

That is sufficient for signalling tests, but not for bearer bridging:

- `dumb/` is not a media-bearing endpoint
- the MGW can accept the call but will not bridge audio
- fixed endpoint-name pairing only works for strict 2-side circuit models

For a shared MGW used by multiple networks, the MGW needs a per-call correlation key, not a static endpoint-pair map.


## Design

Yate now supports an optional bearer-correlation token carried from ISUP into MGCP:

1. `SS7ISUPCall` derives a stable media token from the call's routing label and circuit code.
2. `mgcpca` stores that token on the circuit.
3. If enabled for a given MGCP gateway profile, `mgcpca` sends the token in MGCP as `x-calltoken`.
4. `mgcpgw` watches incoming `CRCX`/`MDCX` requests.
5. When two established MGCP legs arrive with the same `x-calltoken`, `mgcpgw` bridges them locally.

This allows a single MGW to bridge bearer legs for calls between any participating networks, as long as both sides derive the same token for the same ISUP call.


## Token Format

The token is built from:

- SS7 label type
- normalized `OPC`
- normalized `DPC`
- circuit code (`CIC`)

Normalization means:

- `OPC` and `DPC` are sorted so the token is direction-independent
- `SLS` is not used

Current token shape:

`isup/<type>/<min-opc>/<max-opc>/<cic>`

This is intentionally based on call identity, not transport path details.


## Why This Scales

This model works for:

- `A <-> B`
- `A <-> C`
- `B <-> C`
- `A <-> D`
- and so on

without configuring endpoint pairs on the MGW.

The MGW does not need to know in advance which two endpoint names belong together. It only needs the same token on both bearer legs.


## Configuration

Token export is disabled by default.

Enable it per MGCP gateway profile in `mgcpca.conf`:

```ini
[gw shared-mgw]
host=192.168.105.172
user=acgw101
media_token=yes
```

Leave it disabled for hardware or ordinary MGCP gateways:

```ini
[gw hardware-gw]
host=10.0.0.20
user=gw1
; media_token not set
```

This keeps the feature opt-in on a per-gateway basis.


## Interoperability

### Yate MGW

This feature is intended for Yate `mgcpgw`.

If `x-calltoken` is present on both legs and both legs belong to the same call, the MGW bridges them locally.

### Hardware MGW

`media_token=yes` should not be enabled blindly for third-party MGWs.

Reason:

- the token is sent as an MGCP extension parameter (`x-calltoken`)
- compliant peers should ignore unknown extension parameters
- strict or buggy MGWs may reject them

Use `media_token=yes` only on MGW profiles that are known to support this deployment model.


## Effect on SS7

This feature does not change SS7 call-control behavior.

It does not modify:

- `IAM`, `ACM`, `ANM`, `REL`, `RLC`
- circuit reservation logic
- timer behavior
- SS7 message encoding or decoding

It only derives a bearer-correlation token from existing call identity and passes it into MGCP bearer control.


## STP Behavior

Adding a normal STP in the SIGTRAN/SS7 path should not break this model.

Reason:

- the token uses normalized `OPC`, `DPC`, and `CIC`
- it does not use `SLS`
- a normal STP forwards routing and does not redefine call identity

This can break only if an intermediate node rewrites the effective call identity, for example:

- point code translation that changes the `OPC/DPC` seen by the two ends
- a gateway or interworking node that remaps `CIC`
- a B2BUA-like signalling element that terminates and re-originates the call


## Operational Notes

- `dumb/` is useful for signalling tests only. It is not a bearer bridge.
- For shared-MGW bearer bridging, rely on token matching, not endpoint-name pairing.
- The first bearer leg is parked on the MGW until the matching leg arrives.
- When the matching leg arrives, the MGW bridges the two channels locally.
- Unmatched legs continue through normal call routing behavior.


## Troubleshooting

Things to verify in logs:

1. Both sides of the call derive the same ISUP identity.
2. `media_token=yes` is enabled on the `mgcpca` gateway profile that points to the shared MGW.
3. The outgoing `CRCX` contains `x-calltoken`.
4. The MGW receives two bearer legs with the same `x-calltoken`.
5. Audio is not being routed to `dumb/`.

If calls still fail to bridge:

- confirm the two sides see the same effective `OPC`, `DPC`, and `CIC`
- confirm the deployment is using Yate `mgcpgw`
- confirm no intermediate signalling element rewrites the call identity

