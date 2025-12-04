# Fix: Payload Deserializers Trust Counters

## Issue Number
Issue 8 (Medium Severity)

## Description
Multiple UBX payload deserializers iterate based on counter fields read directly from the payload (e.g., `num_svs`, `num_meas`, `num_sv`). The `buf_offset<T>()` utility function performs unchecked `memcpy` operations. If a counter field is corrupted or maliciously crafted, the loop can read far past the end of the payload buffer.

## Evidence
- **File:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/utils.hpp`
  - Lines 193-198: `buf_offset<T>()` performs unchecked memcpy

**Before fix:**
```cpp
template<typename T>
inline T buf_offset(std::vector<u1_t> * buf, u2_t offset)
{
  T value;
  memcpy(&value, buf->data() + offset, sizeof(T));  // No bounds check!
  return value;
}
```

**Affected deserializers (7+ locations):**
- `NavSatPayload` - loops on `num_svs`
- `NavSigPayload` - loops on `num_sigs`
- `NavOrbPayload` - loops on `num_sv`
- `RxmRawxPayload` - loops on `num_meas`
- `RxmMeasxPayload` - loops on `num_sv`
- `SecSigPayload` - loops on `jam_num_cent_freqs`
- `SecSigLogPayload` - loops on `num_events`

**Example vulnerable pattern:**
```cpp
num_svs = buf_offset<u1_t>(&payload_, 5);
for (u1_t i = 0; i < num_svs; ++i) {  // If num_svs is corrupted...
  sat.gnss_id = buf_offset<u1_t>(&payload_, offset);  // ...reads past buffer
  // ...
  offset += 12;
}
```

## Impact
- Out-of-bounds read when counter field is corrupted/malformed
- Crash or memory corruption
- Could be triggered by malformed UBX messages

## Fix Applied
Added bounds checking to `buf_offset<T>()` that throws `UbxPayloadException` on OOB access:

```cpp
template<typename T>
inline T buf_offset(std::vector<u1_t> * buf, u2_t offset)
{
  // FIX Issue 8: Bounds check before memcpy to prevent OOB read
  if (static_cast<size_t>(offset) + sizeof(T) > buf->size()) {
    throw UbxPayloadException(
      "buf_offset: OOB access at offset " + std::to_string(offset) +
      " + " + std::to_string(sizeof(T)) + " > buf size " + std::to_string(buf->size()));
  }
  T value;
  memcpy(&value, buf->data() + offset, sizeof(T));
  return value;
}
```

## Files Changed
- `ublox_dgnss_node/include/ublox_dgnss_node/ubx/utils.hpp`

## How to Verify

### Exception Handling
If a malformed payload is received, the exception will be caught by existing handlers:
```
[ERROR] ubx_queue_frame_in UbxPayloadException: buf_offset: OOB access at offset 156 + 4 > buf size 128
```

### Build with Address Sanitizer
```bash
colcon build --packages-select ublox_dgnss_node --cmake-args -DENABLE_ADDRESS_SANITIZER=ON
```

Before the fix, corrupted counter fields would cause ASan to detect:
```
==12345==ERROR: AddressSanitizer: heap-buffer-overflow
READ of size X at 0x...
```

After the fix, the exception is thrown before any OOB access occurs.

### Benefits
- Single fix protects all 7+ deserializer locations
- Uses existing exception infrastructure (UbxPayloadException)
- Descriptive error messages aid debugging
