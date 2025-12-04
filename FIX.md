# Fix: CFG-VALGET Parser Underflows Size Check

## Issue Number
Issue 6 (Medium Severity)

## Description
The CFG-VALGET payload parser uses an unsigned subtraction in its loop condition that underflows when the payload size is less than 4 bytes. This causes the loop to iterate approximately 18 quintillion times, reading far past the end of the buffer.

## Evidence
- **File:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx_cfg.hpp`

**Before fix (line ~80):**
```cpp
while (idx < static_cast<size_t>(size) - 4) {
  // If size < 4, this becomes:
  // idx < 0xFFFFFFFFFFFFFFFC (huge number due to underflow)
  // Loop runs wild, reading past buffer
}
```

**Also vulnerable (lines 74-76):**
```cpp
version = payload_[0];   // OOB if size < 1
layer = payload_[1];     // OOB if size < 2
position = *reinterpret_cast<u2_t *>(&payload_[2]);  // OOB if size < 4
```

## Impact
- Unsigned integer underflow when `size < 4`
- Massive out-of-bounds read (potentially gigabytes)
- Crash or memory corruption
- Could be triggered by malformed UBX-CFG-VALGET response

## Fix Applied
1. Added guard to return early if `size < 4`
2. Changed loop condition from `idx < size - 4` to `idx + 4 <= size` (no underflow possible)
3. Added bounds check before value extraction
4. Added observation logging (stderr) when undersized payload is detected

```cpp
// FIX Issue 6: Guard against undersized payloads
if (size < 4) {
  std::cerr << "Issue 6 observation: CFG-VALGET payload too small (size=" 
            << size << ", min=4)..." << std::endl;
  version = 0;
  layer = 0;
  position = 0;
  return;
}

// FIX Issue 6: Restructured condition to avoid underflow
while (idx + 4 <= static_cast<size_t>(size)) {
  // ...
  // FIX Issue 6: Bounds check before reading value
  if (idx + value_size > static_cast<size_t>(size)) {
    break;
  }
  // ...
}
```

## Files Changed
- `ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx_cfg.hpp`

## How to Verify

### Observation Logging
If an undersized CFG-VALGET response is received, you will see on stderr:
```
Issue 6 observation: CFG-VALGET payload too small (size=X, min=4). 
Without fix, this would cause underflow/OOB.
```

### Before Fix (demonstrates the bug)
Add this debug code to the original:
```cpp
if (size < 4) {
  std::cerr << "BUG: size=" << size << " would cause underflow!" << std::endl;
  std::cerr << "size - 4 = " << (static_cast<size_t>(size) - 4) << std::endl;
  // Would print something like: size - 4 = 18446744073709551612
}
```

### Build with Address Sanitizer
```bash
colcon build --packages-select ublox_dgnss_node --cmake-args -DENABLE_ADDRESS_SANITIZER=ON
```

Before the fix, ASan would detect massive out-of-bounds reads if this code path was triggered.
