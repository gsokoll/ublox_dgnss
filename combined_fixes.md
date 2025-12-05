# Combined Bug Fixes for ublox_dgnss

This document tracks all bug fixes merged into the `fix_testing` branch.

---
# Fix: UBX Frame Parsing Allows Truncated Packets

## Issue Number
Issue 1 (High Severity)

## Description
The UBX frame parser accepts packets with only 3-7 bytes when detecting UBX sync characters (0xB5 0x62). The `from_buf_build()` function then unconditionally accesses buffer indices 0-5 and reads a 2-byte length field, causing out-of-bounds reads on truncated packets.

## Evidence
- **File:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp`
  - Line ~1453: `if (len > 2 && buf[0] == ubx::UBX_SYNC_CHAR_1 && buf[1] == ubx::UBX_SYNC_CHAR_2)`
  - Line ~1511: Same pattern in `ublox_out_callback`
- **File:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx.hpp`
  - Lines 63-73: `from_buf_build()` accesses `buf[0]` through `buf[5]` without size validation

## Impact
- Out-of-bounds memory read when processing truncated UBX frames
- Potential crash or undefined behavior
- Could be triggered by USB communication errors or malformed data

## Fix Applied
1. Changed the condition from `len > 2` to `len >= 8` to ensure minimum valid UBX frame size
2. A valid UBX frame requires: sync(2) + class(1) + id(1) + length(2) + checksum(2) = 8 bytes minimum
3. Added observation logging (`RCLCPP_WARN`) when truncated frames are rejected

## Files Changed
- `ublox_dgnss_node/src/ublox_dgnss_node.cpp`

## How to Verify

### Before Fix (demonstrates the bug)
Add this check before `from_buf_build()` call on the original code:
```cpp
if (len > 2 && len < 8 && buf[0] == ubx::UBX_SYNC_CHAR_1 && buf[1] == ubx::UBX_SYNC_CHAR_2) {
  RCLCPP_ERROR(get_logger(), "BUG: Truncated UBX frame would cause OOB access (len=%zu)", len);
}
```

### After Fix
Run the node normally. If truncated frames occur, you will see:
```
[WARN] Issue 1 observation: Truncated UBX frame rejected (len=X, min valid=8)...
```
The node will continue running safely instead of potentially crashing.

### Testing with Address Sanitizer
Build with ASan to detect any remaining OOB access:
```bash
colcon build --packages-select ublox_dgnss_node --cmake-args -DENABLE_ADDRESS_SANITIZER=ON
```

---

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
