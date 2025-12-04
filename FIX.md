# Fix: UBX/RTCM Queue Consumers Lack Locks (Diagnostic Branch)

## Issue Number
Issue 2 (High Severity)

## Description
The UBX and RTCM queue consumers read and access queue elements without holding the mutex, while producers push under the mutex. This creates a data race on `std::deque` internals when both threads access the queue concurrently.

## Evidence
- **File:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp`
  - Lines ~1811-1817: `ubx_queue_.size()` and `ubx_queue_[0]` accessed without lock
  - Lines ~1835-1838: Only `pop_front()` is guarded by mutex
  - Lines ~1859-1865: Same pattern for `rtcm_queue_`

**Producer (with lock):**
```cpp
{
  const std::lock_guard<std::mutex> lock(ubx_queue_mutex_);
  ubx_queue_.push_back(queue_frame);
}
```

**Consumer (NO lock on read):**
```cpp
while (ubx_queue_.size() > 0) {
  ubx_queue_frame_t f = ubx_queue_[0];  // DATA RACE
  // ... process ...
  {
    const std::lock_guard<std::mutex> lock(ubx_queue_mutex_);
    ubx_queue_.pop_front();  // only pop is locked
  }
}
```

## Impact
- Data race on `std::deque` internals (undefined behavior per C++ standard)
- Potential crashes under high message load
- Memory corruption if deque reallocates during concurrent access

## Current Status
**This branch contains diagnostic tooling only.** The actual fix has not been implemented pending confirmation of the race via Thread Sanitizer.

### Diagnostic Added
- CMake option `ENABLE_THREAD_SANITIZER` to build with `-fsanitize=thread`

### Proposed Fix (to be implemented after verification)
Guard the entire read-process-pop sequence, or copy front element under lock:
```cpp
ubx_queue_frame_t f;
{
  const std::lock_guard<std::mutex> lock(ubx_queue_mutex_);
  if (ubx_queue_.empty()) break;
  f = ubx_queue_.front();
  ubx_queue_.pop_front();
}
// process f outside lock
```

## Files Changed
- `ublox_dgnss_node/CMakeLists.txt` (added ENABLE_THREAD_SANITIZER option)

## How to Verify

### Build with Thread Sanitizer
```bash
colcon build --packages-select ublox_dgnss_node --cmake-args -DENABLE_THREAD_SANITIZER=ON
```

### Run and observe
```bash
source install/setup.bash
ros2 run ublox_dgnss_node ublox_dgnss_node
```

### Expected TSan output if race occurs
```
WARNING: ThreadSanitizer: data race
  Write of size 8 at 0x... by thread T1:
    #0 std::deque::push_back() ...
    #1 ublox_in_callback() ublox_dgnss_node.cpp:...

  Previous read of size 8 at 0x... by thread T2:
    #0 std::deque::operator[]() ...
    #1 ubx_timer_callback() ublox_dgnss_node.cpp:...
```

### Notes
- Thread Sanitizer introduces ~10x slowdown
- The race is timing-dependent; may need extended runtime or high message rate to trigger
- Only works on Linux with GCC/Clang
