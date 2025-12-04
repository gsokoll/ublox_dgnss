# Fix: Shared Flags Without Synchronization

## Issue Number
Issue 9 (Low Severity)

## Description
The `keep_running_` and `attached_` flags in the USB Connection class are plain `bool` variables accessed from multiple threads without synchronization. This constitutes a data race under the C++ memory model, which is undefined behavior. While x86 architectures often hide this issue due to strong memory ordering, it's formally incorrect and could cause problems on other architectures or with aggressive compiler optimizations.

## Evidence
- **File:** `ublox_dgnss_node/include/ublox_dgnss_node/usb.hpp`
  - Lines 144-145: `bool keep_running_;` and `bool attached_;`

- **File:** `ublox_dgnss_node/src/usb.cpp`
  - Line 46: `keep_running_ = true;` (initialization in constructor)
  - Line 480: `attached_ = true;` (set in hotplug callback - different thread)
  - Line 500: `attached_ = false;` (set in hotplug callback)
  - Line 921: `if (!keep_running_)` (read in event loop - different thread)
  - Line 934: `attached_ = false;` (set in event handler)

**Thread access pattern:**
- Main thread: reads/writes `keep_running_`, `attached_`
- libusb event thread: reads/writes `keep_running_`, `attached_`
- Hotplug callback thread: writes `attached_`

## Impact
- Undefined behavior per C++ standard (data race)
- Could cause missed updates or stale reads
- May cause issues on weakly-ordered architectures (ARM, etc.)
- Compiler could optimize in unexpected ways

## Fix Applied
Changed both flags from `bool` to `std::atomic<bool>`:

```cpp
// FIX Issue 9: Use atomic for cross-thread flags to avoid data race
std::atomic<bool> keep_running_;
std::atomic<bool> attached_;
```

No changes required to usage code - `std::atomic<bool>` supports the same assignment and comparison operators as plain `bool`.

## Files Changed
- `ublox_dgnss_node/include/ublox_dgnss_node/usb.hpp`

## How to Verify

### Build with Thread Sanitizer
```bash
colcon build --packages-select ublox_dgnss_node --cmake-args -DENABLE_THREAD_SANITIZER=ON
```

### Before Fix
TSan reports data races:
```
WARNING: ThreadSanitizer: data race
  Write of size 1 at 0x... by thread T1:
    #0 Connection::hotplug_attach_callback() usb.cpp:480
  Previous read of size 1 at 0x... by main thread:
    #0 Connection::handle_events() usb.cpp:921
```

### After Fix
No data race warnings from TSan.

### Notes
- `std::atomic<bool>` has minimal performance overhead on x86
- Provides correct memory ordering guarantees
- Makes the code portable to other architectures
