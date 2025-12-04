# Fix: Exception Safety of user_data Allocation

## Issue Number
Issue 10 (Low Severity)

## Description
The `make_transfer_out()` and `make_transfer_in()` functions allocate a heap object using `new` and assign it to `transfer->usb_transfer->user_data`. If any code between the allocation and successful function return throws an exception, the allocated memory leaks because there's no RAII wrapper to clean it up.

While the current code path doesn't throw (libusb functions are C and don't throw C++ exceptions), this is poor defensive coding practice that could cause issues if future changes introduce throwing code.

## Evidence
- **File:** `ublox_dgnss_node/src/usb.cpp`

**make_transfer_out (before fix):**
```cpp
transfer->usb_transfer->user_data = new std::shared_ptr<transfer_t>(transfer);
transfer->usb_transfer->flags = LIBUSB_TRANSFER_SHORT_NOT_OK;

libusb_fill_bulk_transfer(...);
// If anything throws before return, the 'new' allocation leaks
return transfer;
```

**make_transfer_in (before fix):**
```cpp
transfer->usb_transfer->user_data = new std::shared_ptr<transfer_t>(transfer);

libusb_fill_bulk_transfer(...);
// If anything throws before return, the 'new' allocation leaks
return transfer;
```

## Impact
- Memory leak if exception occurs during transfer setup
- Currently theoretical (C libusb functions don't throw)
- Violates RAII principles and defensive coding practices

## Fix Applied
Use `std::unique_ptr` to manage the allocation, and `release()` ownership only after setup is complete:

```cpp
// FIX Issue 10: Use unique_ptr for exception safety, release only after setup complete
auto user_data_ptr = std::make_unique<std::shared_ptr<transfer_t>>(transfer);

libusb_fill_bulk_transfer(
  transfer->usb_transfer, devh_, ...,
  callback_fn, user_data_ptr.get(), 0  // Pass raw pointer to libusb
);

// Transfer ownership to libusb - caller responsible for cleanup via callback
transfer->usb_transfer->user_data = user_data_ptr.release();

return transfer;
```

**How it works:**
1. `make_unique` creates a `unique_ptr` managing the heap allocation
2. If any exception occurs before `release()`, the `unique_ptr` destructor frees the memory
3. `release()` transfers ownership to libusb only after all setup succeeds
4. Callback is still responsible for cleanup (unchanged from before)

## Files Changed
- `ublox_dgnss_node/src/usb.cpp`

## How to Verify

### Code Review
The fix is primarily about code hygiene and future-proofing. To verify:
1. Review that `unique_ptr` is created before any potentially-throwing code
2. Verify `release()` is called only at the end after all setup
3. Confirm callback cleanup code is unchanged

### Testing
Normal operation should be unchanged. The fix is defensive and only activates if an exception occurs during setup.

### Theoretical Test
To test the exception safety, you could temporarily add a throwing statement between the allocation and release:
```cpp
auto user_data_ptr = std::make_unique<std::shared_ptr<transfer_t>>(transfer);
throw std::runtime_error("test");  // unique_ptr will clean up
transfer->usb_transfer->user_data = user_data_ptr.release();
```

Without the fix, this would leak. With the fix, the memory is properly freed.
