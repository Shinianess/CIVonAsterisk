# CIV (Caller Identification Verification) on Asterisk

This project implements a **CIV** (Caller ID Verification) scheme to prevent Caller ID spoofing. It uses a challenge‑response mechanism to verify whether the caller is the owner of the claimed callerID. The callee's endpoint (stock SIP client) receives either `Verified CallerID` or `Spoofed CallerID` as the verification result.

---

## System Architecture & Verification Flow

The flow involves two Asterisk instances:

- **Caller side** – the originating Asterisk (configuration in `caller's Asterisk.conf`).
- **Callee side** – the terminating Asterisk (configuration in `callee's Asterisk.conf`).

### Sequence Diagram


```
             Alice's carrier          Bob's carrier
          gateway (serverfr)       gateway (serverca)
                      ┌─┐                    ┌─┐
                      │ │ 2. calls Bob from  │ │
                      │ │  Alice's number    │ │
          1. calls Bob│ │───────────────────▶│ │ 5. rings from
  ┌───────┐ as Alice  │ │ 3. sends challenge │ │ Alice's number┌───────┐
  │Alice's│──────────▶│ │  to Alice's number │ │──────────────▶│ Bob's │
  │ phone │           │ │◀───────────────────│ │(authenticated)│ phone │
  └───────┘           │ │                    │ │               └───────┘
                      │ │ 4. sends response  │ │
                      │ │──────────────────▶ │ │
                      └─┘                    └─┘
```

         Figure 1: authenticated caller with an unmodified number


```
             Alice's carrier          Bob's carrier
          gateway (serverfr)       gateway (serverca)
                    ┌─┐                    ┌─┐
                    │ │ 2. calls Bob from  │ │
                    │ │  Alice's number    │ │
        1. calls Bob│ │───────────────────▶│ │
┌───────┐ as Alice  │ │ 3. sends challenge │ │
│Alice's│──────────▶│ │  to Alice's number │ │     ┌───────┐
│ phone │           │ │◀───────────────────│ │     │ Bob's │
└───────┘           │ │                    │ │     │phone 1│    Bob's carrier 2
                    │ │ 4. sends response  │ │     └───────┘  gateway (serversg)
                    │ │──────────────────▶ │ │                       ┌─┐
                    │ │                    │ │ 5. forwards to Bob's  │ │
                    │ │                    │ │      2nd number       │ │
                    │ │                    │ │──────────────────────▶│ │
                    │ │                    │ │   6. sends challenge  │ │
                    │ │                    │ │   to Alice's number   │ │
                    │ │◀───────────────────┼─┼────────────────────── │ │
                    │ │                    │ │                       │ │
                    │ │ 7. sends response  │ │ 8. forwards response  │ │
                    │ │───────────────────▶│ │──────────────────────▶│ │
                    │ │                    │ │                       │ │ 9. rings from  ┌───────┐
                    │ │                    │ │                       │ │ Alice's number │ Bob's │
                    │ │                    │ │                       │ │ ──────────────▶│phone 2│
                    └─┘                    └─┘                       └─┘ (authenticated)└───────┘
```


         Figure 2: Call forwarding based on proxy forwarding
---

## Configuration Steps

### Prerequisites

- Tested on Asterisk 23 (with `chan_pjsip` and `PJSIP_HEADER` function)
- Properly configured SIP peers (`serverca`, `serverfr` and `serversg`) with routing
- DTMF mode set (RFC 4733 recommended)

### Caller‑Side Asterisk

Copy the code from `caller's Asterisk.conf` into your `extensions.conf`.

**Tips**:
- Replace `serverca` with your callee’s SIP peer name that routes the called number.
- Replace `serversg` with your spoofer’s SIP peer name.
- The extension `3922618` is the caller's number. Please create and register the endpoint before running the code.
- The number '392231' is the callee's number. By default, dialling extension `9392231` with CIV enabled, while '9312293' disable CIV. The callee's endpoint will show the verification result accordingly.
- Dialling '93922800' enables server-based forwarding. The callee's server (serverca) will forward the call to 3922700@serversg. The CIV result will show on the endpoint registered as 3922700 on serversg.

### Callee‑Side Asterisk

Copy the code from `callee's Asterisk.conf` into your `extensions.conf`.

**Tips**:
- Replace `serverfr` with the callee’s SIP peer name used for callback routing.
- Replace `serversg` with your spoofer’s SIP peer name.
- `392231` is the final callee extension; adjust as needed.

### Spoofer‑Side Asterisk

Copy the code from `callee's Asterisk.conf` into your `extensions.conf`.

**Tips**:
- Replace `serverfr` with the callee’s SIP peer name used for callback routing.
- Replace `serverca` with your callee’s SIP peer name that routes the called number.
- When making call, `3922700` change it callID to '3922618' to spoof the number of the caller. 
- When receiving call, `3922700` is the number to receive the forwarded call from 392231@serverca.


---

## Key Variables in the Dialplan

| Variable | Purpose |
|----------|---------|
| `initial_session_id` | Unique 32‑hex session ID generated by the caller to tie the whole verification together. |
| `CIV_SESSION` | Alias for `initial_session_id`; passed as a channel variable. |
| `callback_cid` | 4‑digit random challenge code generated by the callee. |
| `verif_session_id` | Temporary session ID generated by the callee for the callback call. |
| `DTMF` | The code received via `Read()` (sent back by the caller via `SendDTMF`). |
| `verify_result` | Final string: `Verified CallerID` or `Spoofed CallerID`. |

---

## AstDB Keys Used

- `civ/session/${initial_session_id}` → `active` (marks a valid session)
- `civ/status/${initial_session_id}` → `initial` / `code_received` (state machine)
- `civ/code/${initial_session_id}` → stores the 4‑digit code (written by the callback, read by the caller)

All keys are deleted after a successful verification.

---

## Usage

1. **Caller dials** `9392231` (or your custom number) to initiate a CIV enabled call.
2. **Callee processes** the incoming call:
   - If the SIP headers contain `Supported:civ` and a `Session-ID`, the verification flow is triggered.
   - Otherwise, the call is routed normally with `CALLERID(name)=Unverifiable callerID`.
3. **Verification succeeds** – the receiver's server sets `CALLERID(name)=Verified callerID` and bridges to the final extension.
4. **Verification fails** (DTMF mismatch or timeout) – `CALLERID(name)=Spoofed callerID`.

---

## Troubleshooting & Notes

### DTMF Tuning
- Caller uses `SendDTMF(${code}#,50,100,,)` (duration=100ms, gap=50ms). If DTMF is not recognised reliably, increase both (e.g., `100,100`).
- Callee uses `Read(DTMF,,,,,2)` with a 2‑second timeout; adjust if network latency is high.

### Dependencies
- Ensure `func_odbc` or `func_db` is loaded (`module show like db`).
- Ensure `chan_pjsip` is loaded and supports `PJSIP_HEADER`.
- Verify that the tunnels of `serverca` and `serverfr` are correctly defined in `pjsip.conf`.

### Debugging
- All key steps log timestamps with `Verbose()`. Watch the console (`asterisk -rvvv`) to follow the flow.
- Check database entries: `database show civ`.

### Security Considerations
- The challenge code is 4 digits (change by adjusting `RAND(1000,9999)`).
- Session IDs are UUID‑based and unique.
- DB entries are deleted after use to avoid clutter.

---

## Further work

The current implementation of CIV relies on the Asterisk dialplan. While functional, our long-term objective is to integrate the protocol natively within the Asterisk core. By embedding it directly into the source code, CIV will function as a built-in service, improving performance, reliability, and ease of deployment.
