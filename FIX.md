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
