# Fix: Use-After-Free During Transfer Cancellation (Diagnostic Branch)

## Issue Number
Issue 3 (High Severity)

## Description
The `cleanup_all_transfers()` function cancels libusb transfers and then immediately clears the transfer queue. However, `libusb_cancel_transfer()` is asynchronous — callbacks fire later via `libusb_handle_events()`. By clearing the queue immediately, the `transfer_t` objects (and their `user_data`) are freed while libusb still holds pointers to them, causing use-after-free when callbacks eventually run.

## Evidence
- **File:** `ublox_dgnss_node/src/usb.cpp`
  - Lines ~856-858: `libusb_cancel_transfer()` called (asynchronous)
  - Line ~867: `transfer_queue_.clear()` called immediately after
  - Lines ~618-620, ~680-682: Callbacks dereference `user_data` which points to freed memory

```cpp
// Cancel all active transfers
for (auto & transfer : transfer_queue_) {
  if (!transfer->completed && transfer->usb_transfer) {
    int rc = libusb_cancel_transfer(transfer->usb_transfer);  // async!
    ...
  }
}
// Clear the entire queue
transfer_queue_.clear();  // FREES transfer_t objects immediately!
```

Later, when callbacks fire:
```cpp
auto sp = static_cast<std::shared_ptr<transfer_t> *>(transfer->user_data);  // UAF!
(*sp)->completed = true;
delete sp;
```

## Impact
- Use-after-free when callbacks run after queue is cleared
- Crash during device disconnect or node shutdown
- Memory corruption and undefined behavior

## Current Status
**This branch contains diagnostic tooling only.** The actual fix has not been implemented pending confirmation via Address Sanitizer.

### Diagnostic Added
- CMake option `ENABLE_ADDRESS_SANITIZER` to build with `-fsanitize=address`

### Proposed Fix (to be implemented after verification)
After cancelling transfers, continue calling `libusb_handle_events()` until all callbacks have run and marked transfers as `completed`, then clear the queue:
```cpp
// Cancel all transfers
for (auto & transfer : transfer_queue_) {
  if (!transfer->completed && transfer->usb_transfer) {
    libusb_cancel_transfer(transfer->usb_transfer);
  }
}

// Wait for all callbacks to complete
while (has_pending_transfers()) {
  libusb_handle_events_timeout(ctx_, &timeout);
}

// Now safe to clear
transfer_queue_.clear();
```

## Files Changed
- `ublox_dgnss_node/CMakeLists.txt` (added ENABLE_ADDRESS_SANITIZER option)

## How to Verify

### Build with Address Sanitizer
```bash
colcon build --packages-select ublox_dgnss_node --cmake-args -DENABLE_ADDRESS_SANITIZER=ON
```

### Trigger condition
1. Run the node with a connected USB device
2. Disconnect the USB device (or shutdown the node) while transfers are in flight

### Expected ASan output if UAF occurs
```
==12345==ERROR: AddressSanitizer: heap-use-after-free
READ of size 8 at 0x... thread T1
    #0 ... callback_in() usb.cpp:680
    ...
freed by thread T2 here:
    #0 ... cleanup_all_transfers() usb.cpp:867
```

### Notes
- Address Sanitizer introduces ~2x slowdown
- Only works on Linux with GCC/Clang
