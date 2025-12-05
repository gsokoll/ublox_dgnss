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
