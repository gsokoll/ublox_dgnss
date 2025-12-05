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
