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

---
# Fix: NMEA Logging Writes Past Buffer End

## Issue Number
Issue 4 (High Severity)

## Description
When logging NMEA strings, the code writes a null terminator at `buf[len]` to convert the buffer to a C-string. However, `buf` points to a libusb transfer buffer of fixed size (`IN_BUFFER_SIZE = 12800`), and `len` is the actual received length. If `actual_length == buffer_length`, writing at index `len` is a 1-byte buffer overrun.

## Evidence
- **File:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp`
  - Line ~1444 (before fix): `buf[len] = 0;`

```cpp
size_t len = transfer_in->actual_length;
unsigned char * buf = transfer_in->buffer;

if (buf[0] == 0x24) {  // NMEA starts with '$'
  buf[len] = 0;  // OOB WRITE if len == buffer_length!
  // ... strip trailing \r\n ...
  RCLCPP_INFO(get_logger(), "nmea: %s", buf);
}
```

- **File:** `ublox_dgnss_node/include/ublox_dgnss_node/usb.hpp`
  - Line ~48: `#define IN_BUFFER_SIZE 64 * 200` (12800 bytes)

## Impact
- 1-byte heap buffer overflow when NMEA message fills entire transfer buffer
- Memory corruption
- Potential crash or security vulnerability

## Fix Applied
1. Copy NMEA data to a local `std::string` instead of modifying the libusb buffer directly
2. Use `std::string::pop_back()` to strip trailing `\r\n` characters safely
3. Added observation logging when buffer-full condition is detected

```cpp
// FIX: Copy to local string instead of writing into libusb buffer
std::string nmea_str(reinterpret_cast<char*>(buf), len);
// Strip trailing \r\n
while (!nmea_str.empty() && 
       (nmea_str.back() == '\r' || nmea_str.back() == '\n')) {
  nmea_str.pop_back();
}
RCLCPP_INFO(get_logger(), "nmea: %s", nmea_str.c_str());
```

## Files Changed
- `ublox_dgnss_node/src/ublox_dgnss_node.cpp`

## How to Verify

### Observation Logging
If the buffer-full condition occurs, you will see:
```
[WARN] Issue 4 observation: NMEA buffer full (len=12800, buf_size=12800). 
       Without fix, this would write 1 byte past buffer end.
```

### Build with Address Sanitizer
```bash
colcon build --packages-select ublox_dgnss_node --cmake-args -DENABLE_ADDRESS_SANITIZER=ON
```

Run the node. Before the fix, ASan would detect:
```
==12345==ERROR: AddressSanitizer: heap-buffer-overflow
WRITE of size 1 at 0x...
```

After the fix, no buffer overflow occurs.

### Additional Benefits
- Cleaner code using modern C++ string operations
- Removed unused `remove_any_of` variable

---

# Fix: Missing Exception Handling in RTCM Callback Causes Crash

## Issue Number
Issue 12 (High Severity)

## Description
When a USB timeout or error occurs during RTCM data transmission, the `write_buffer()` function throws a `usb::UsbException`. This exception is not caught in `rtcm_callback()`, causing `std::terminate()` to be called and crashing the entire node.

## Evidence
From crash logs:
```
terminate called after throwing an instance of 'usb::UsbException'
  what(): Error while sending buf: LIBUSB_ERROR_TIMEOUT
[ERROR] process has died [pid 386571, exit code -6, ...]
```

Exit code -6 indicates SIGABRT from an uncaught exception.

## Root Cause
In `ublox_dgnss_node.cpp`, the `rtcm_callback()` function calls `usbc_->write_buffer()` without any exception handling:

```cpp
void rtcm_callback(const rtcm_msgs::msg::Message & msg) const
{
    // ... validation checks ...
    usbc_->write_buffer(data_out.data(), data_out.size());  // Can throw!
}
```

When `write_buffer()` encounters a USB error (timeout, device disconnect, etc.), it throws `UsbException`. Since there's no try-catch, the exception propagates up through the ROS callback mechanism, eventually hitting `std::terminate()`.

## Impact
- Node crashes on transient USB errors during RTCM transmission
- Loss of GPS service until manual restart
- Particularly problematic during brief USB communication glitches that would otherwise be recoverable

## Fix Applied
1. Wrapped `write_buffer()` call in `rtcm_callback()` with try-catch
2. Log error message instead of crashing
3. Also made exception handling explicit in `get_polled_frame()` in `ubx.hpp` for consistency

## Files Changed
- `ublox_dgnss_node/src/ublox_dgnss_node.cpp` - Added try-catch in rtcm_callback
- `ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx.hpp` - Explicit exception handling in get_polled_frame

## How to Verify

### Before Fix
Run the node and simulate USB timeout/error during RTCM transmission:
```
terminate called after throwing an instance of 'usb::UsbException'
  what(): Error while sending buf: LIBUSB_ERROR_TIMEOUT
```
Node crashes with exit code -6.

### After Fix
Same scenario produces:
```
[ERROR] Failed to send RTCM data: Error while sending buf: LIBUSB_ERROR_TIMEOUT
```
Node continues running, attempting to recover on next RTCM message.

## Additional Notes
This fix prevents crashes but does not address the underlying cause of USB timeouts. If timeouts are frequent, there may be:
- USB hardware/cable issues
- System resource contention
- Need for Issue #13 (mutex protection on write_buffer) to prevent race conditions

Related issues:
- Issue #13: Missing mutex on write_buffer (thread safety)
- Issue #5: libusb callback leak on errors (memory management)
