# Fix: CFG-VALGET NAK Handling Causes Parameter Fetch Timeout

## Issue Number
Issue 11 (High Severity)

## Description
When the u-blox device NAKs a CFG-VALGET request (e.g., because a parameter is not supported by the device's firmware version), the driver does not handle the NAK properly. Parameters that were marked as `PARAM_VALGET` (waiting for response) remain stuck in that state indefinitely, causing a 5-second timeout and "degraded mode" initialization.

## Evidence
From logs:
```
[WARN] ubx class: 0x05 id: 0x00 ack nak payload - class: 0x06 id: 0x8b
...
[ERROR] Parameter fetch TIMEOUT after 5 seconds! 70 parameters stuck in PARAM_VALGET:
[ERROR]   Missing response: CFG_INFMSG_NMEA_USB
[ERROR]   Missing response: CFG_INFMSG_UBX_USB
...
```

The NAK payload shows:
- `class: 0x06` = UBX-CFG
- `id: 0x8b` = CFG-VALGET

This indicates the device rejected the VALGET request, but the code only logged a warning and did not transition the affected parameters out of `PARAM_VALGET` state.

## Root Cause
In `ubx_ack_frame()`, when a NAK is received for CFG-VALGET:
1. The code logs a warning
2. But does NOT clear the pending parameters from `PARAM_VALGET` status
3. The timeout check waits for responses that will never come

## Impact
- 5-second delay during initialization
- "Degraded mode" warning even when device is functioning correctly
- Potential issues with parameter state tracking
- May mask real problems vs firmware incompatibility

## Fix Applied
1. Added tracking of pending VALGET batches (`pending_valget_batches_` deque)
2. When sending VALGET requests, record which parameters are in each batch
3. When a NAK is received for CFG-VALGET, pop the oldest pending batch and transition those parameters to `PARAM_ACKNAK` status
4. When a successful VALGET response is received, also pop the pending batch
5. Updated completion check to report NAK'd parameters separately

Note: `cfg_val_get_poll_async_all_layers()` sends 2 requests per batch (default layer + RAM layer), so each batch is tracked twice.

## Files Changed
- `ublox_dgnss_node/src/ublox_dgnss_node.cpp`

## How to Verify

### Before Fix
Run the node with a device that doesn't support all parameters. You will see:
```
[ERROR] Parameter fetch TIMEOUT after 5 seconds! XX parameters stuck in PARAM_VALGET:
[WARN] Proceeding with partial parameter initialization (degraded mode)
```

### After Fix
Run the node. If some parameters are NAK'd, you will see:
```
[WARN] Issue 11: CFG-VALGET NAK received, clearing X parameters from pending state
[WARN] Parameter fetch completed - X loaded, Y rejected by device (unsupported)
```

The node will complete initialization quickly without the 5-second timeout.

## Additional Notes
Parameters that are NAK'd (transitioned to `PARAM_ACKNAK`) are likely unsupported by your device's firmware version. This is expected behavior for firmware version mismatches and should not cause operational issues - the device simply doesn't support those configuration options.
