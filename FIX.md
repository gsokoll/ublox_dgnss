# Fix: libusb Callbacks Leak on Error/Disconnect

## Issue Number
Issue 5 (Medium Severity)

## Description
The libusb callback functions (`callback_out` and `callback_in`) have early `return` statements for certain error conditions (TRANSFER_ERROR, NO_DEVICE). These early returns skip the cleanup code that deletes the heap-allocated `shared_ptr` stored in `transfer->user_data`, causing memory leaks.

## Evidence
- **File:** `ublox_dgnss_node/src/usb.cpp`

**callback_out (before fix):**
```cpp
case LIBUSB_TRANSFER_ERROR:
  msg = "Transfer failed";
  return;  // LEAK: skips cleanup at lines 618-620
  break;
// ...
case LIBUSB_TRANSFER_NO_DEVICE:
  msg = "Transfer device disconnected";
  return;  // LEAK: skips cleanup at lines 618-620
  break;
```

**callback_in (before fix):**
```cpp
case LIBUSB_TRANSFER_NO_DEVICE:
  msg = "Transfer device disconnected";
  return;  // LEAK: skips cleanup at lines 680-682
  break;
```

**Cleanup code that was being skipped:**
```cpp
auto sp = static_cast<std::shared_ptr<transfer_t> *>(transfer->user_data);
(*sp)->completed = true;
delete sp;  // cleanup
```

## Impact
- Memory leak every time a transfer error or device disconnect occurs
- `transfer_t` objects remain in `transfer_queue_` as stale entries
- Memory usage grows over time with repeated connect/disconnect cycles

## Fix Applied
Removed the early `return` statements so all error paths fall through to the cleanup code:

```cpp
case LIBUSB_TRANSFER_ERROR:
  msg = "Transfer failed";
  // FIX Issue 5: Removed early return that skipped cleanup (memory leak)
  break;
// ...
case LIBUSB_TRANSFER_NO_DEVICE:
  msg = "Transfer device disconnected";
  // FIX Issue 5: Removed early return that skipped cleanup (memory leak)
  break;
```

## Files Changed
- `ublox_dgnss_node/src/usb.cpp`

## How to Verify

### Build with Address Sanitizer (leak detection)
```bash
colcon build --packages-select ublox_dgnss_node --cmake-args -DENABLE_ADDRESS_SANITIZER=ON
```

### Test procedure
1. Run the node with a connected USB device
2. Disconnect the device
3. Exit the node (Ctrl+C)

### Before Fix
ASan reports memory leaks on exit:
```
==12345==ERROR: LeakSanitizer: detected memory leaks
Direct leak of X bytes in Y objects...
```

### After Fix
Clean exit with no leak reports.

### Alternative: Valgrind
```bash
valgrind --leak-check=full ros2 run ublox_dgnss_node ublox_dgnss_node
```
