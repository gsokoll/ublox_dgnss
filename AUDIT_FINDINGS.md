# ublox_dgnss — USB-layer / whole-package audit findings

**Target:** `aussierobots/ublox_dgnss` current `main` **v0.7.4** (clone `gsokoll/ublox_dgnss`, branch `usb-stability`).  
**Date:** 2026-06-16.  **Deliverable:** findings report + candidate patches on this branch (no upstream PRs).

**Method:** graphify structural map of the package → 5 parallel area-audit agents (USB transport, node integration, UBX framing, parameter/config, lifecycle/shutdown) producing findings → one adversarial verifier per finding (refute-by-default, re-reads the cited code) → only confirmed findings kept. **29 of 34 candidate findings confirmed** (2×P0, 11×P1, 13×P2, 3×P3). 39 agents, ~1.08M tokens.

> Severity scale used by the verifiers: **P0** = causes the production wedge or memory corruption/crash; **P1** = race that can cause data loss/hang under realistic conditions; **P2** = robustness/resilience or diagnosability; **P3** = minor. Where this differs from the operational priority in the bring-up plan, both framings are noted below.

## 1. Reconciliation with the bring-up plan priorities

| Plan item | Audit findings | Status |
|---|---|---|
| **P0 — USB wedge recovery** (persistent `LIBUSB_ERROR_TIMEOUT`, no reset path) | USB-01 (no recovery on persistent timeout), USB-02 (streak only counts `NO_DEVICE`), USB-09 (err_count muting hides persistent IN errors) | **Confirmed.** Verifiers scored this cluster P1 on a *memory-safety* scale; operationally it is your **P0** — the multi-hour outage. The standalone `libusb_reset_device()` you proved recovers the device is the escalation step this cluster lacks. |
| **P0 — (memory safety) RTCM write race** | USB-04 / NODE-01 / LIFE-07 (triple-confirmed), plus USB-03 / NODE-04 (non-atomic `devh_`/`attached_`) | **Confirmed P0.** Real use-after-free / null-deref of `devh_` during an ordinary hotplug detach while RTCM streams (normal RTK mode), uncaught by the `UsbException` handler. The audit's hardest finding. |
| **P1 — RTCM write-path safety** (blocking 250 ms, unsynchronized `attached_`/`devh_`) | NODE-02 (250 ms blocking write on subscription thread), USB-03 / NODE-04 (data race) | **Confirmed.** |
| **P2 — atomic-batch CFG-VALSET fragility** | PARAM-01 (one bad key NAKs whole batch, no per-key log/bisect) | **Confirmed.** Fix = bisect-on-NAK + log offending key by name. |
| **Shutdown heap corruption** (`malloc_consolidate unaligned fastbin`, SIGABRT) | LIFE-02 (10 ms event timer reaps transfers / runs callbacks concurrently with shutdown cleanup + buffer free) | **Root cause identified** — use-after-free / double-free in the teardown ordering. |
| **RXM-SFRBX (0x02 0x13) decode gap** ("unknown … doing nothing") | UBX-05 (only first frame per transfer parsed), UBX-04 (SFRBX decoder over-read) | **Reconciled / refuted-as-stated.** An SFRBX decoder *does* exist in 0.7.4 (`ubx/rxm/ubx_rxm_sfrbx.hpp`). The "doing nothing" symptom is better explained by **UBX-05**: `ublox_in_callback` treats one USB bulk transfer as exactly one UBX frame, so SFRBX/RAWX packed *after* NAV-PVT in the same transfer are silently dropped. |

## 2. Canonical bug clusters (dedup map)

Several findings are the same underlying bug seen from different areas. Patches should target the **cluster**, not each id.

| Cluster | Sev | Finding ids | Essence |
|---|---|---|---|
| **A — RTCM teardown race (UAF of `devh_`)** | P0 | USB-04, NODE-01, LIFE-07, NODE-02 | `write_buffer` dereferences `devh_` under `write_mutex_`, but `close_devh()` nulls/closes `devh_` with **no** `write_mutex_`; the lock-free `dev_valid()`/`attached()` pre-check in `rtcm_callback` doesn't protect it. Also 250 ms blocking write on the subscription thread. |
| **B — non-atomic shared state across 3 threads** | P1 | USB-03, NODE-04 | `devh_`, `attached_`, `driver_state_`, transfer-queue read/written by subscription thread, 10 ms libusb-event timer thread, and hotplug callbacks with no synchronization. |
| **C — wedge: no stall recovery** | P0 (ops) / P1 | USB-01, USB-02, USB-09 | Async IN/OUT transfers have no finite timeout; persistent silent timeouts never flip `attached_` (streak only counts `NO_DEVICE`); no `clear_halt`/`reset_device`/watchdog. Matches the multi-hour wedge. |
| **D — shutdown teardown ordering (heap corruption)** | P1 | LIFE-02, LIFE-03 | 10 ms event timer keeps reaping transfers and running callbacks while shutdown cancels/frees buffers and closes the handle → UAF/double-free (`malloc_consolidate`). |
| **E — UBX framing / bounds** | P1/P2 | UBX-05, UBX-01, UBX-02, UBX-03, UBX-04, UBX-08, UBX-07, UBX-06 | One-frame-per-transfer assumption drops packed/partial frames; multiple decoders trust wire length fields and over-read; polled-frame checksum uses AND instead of OR. |
| **F — hotplug/reinit on the event thread** | P1/P2 | NODE-03, USB-08, NODE-06 | Hotplug attach/detach + full re-init run inline on the libusb-event pump thread; attach can re-enter `open_device` unlocked. |
| **G — CFG-VALSET batch fragility + VALGET races** | P2 | PARAM-01, PARAM-02, PARAM-03 | Atomic batch NAK disables all config with no per-key diagnosis; VALGET response parsed without `cfg_batch_mutex_`; completion heuristic leaves unsupported keys stuck. |
| **H — NTRIP destructor hang** | P2 | LIFE-05 | `~NTRIPClientNode` `join()`s a thread blocked in `curl_easy_perform` with no abort. |
| **I — dead/dangerous sync read** | P3 | USB-10, USB-09 | Synchronous `read_chars` path throws `TimeoutException` with no recovery; currently only in commented-out code. |

## 3. Recommended patch order (for `usb-stability`)

1. **Cluster A — serialize device teardown vs writers** (highest value, smallest change). Make `close_devh()`/`hotplug_detach_callback` take `write_mutex_` before `libusb_close`/`devh_=nullptr`; have `write_buffer` re-validate `devh_ != nullptr && attached_` *inside* the lock; make `attached_`/`devh_` access consistent (one lock or atomics). Prefer the async write path so the subscription thread never blocks 250 ms. Fixes USB-04/NODE-01/LIFE-07/NODE-02 and most of B.
2. **Cluster C — stall-recovery state machine (the wedge / your P0).** Give async IN transfers a finite timeout in `libusb_fill_bulk_transfer`; on STALL `libusb_clear_halt` then resubmit; add a watchdog timestamp updated on each completed IN transfer; on no-progress escalate to `libusb_reset_device` + reopen (the move you proved recovers the F9R); make the streak/escalation cover TIMEOUT, not just `NO_DEVICE`.
3. **Cluster D — fix shutdown ordering (heap corruption).** Order: stop the 10 ms timer → ensure no event callback in flight (drain) → cancel transfers → `handle_events` to reap → free buffers → `libusb_close` (guarded). Fixes the `malloc_consolidate` SIGABRT.
4. **Cluster E — framing.** Implement a persistent byte-stream accumulator in `ublox_in_callback` (handle multiple/partial frames per transfer) — this also fixes the SFRBX/RAWX "doing nothing". Add payload-length bounds checks to the RXM-RAWX/SFRBX/NMEA decoders; fix the polled-frame checksum AND→OR.
5. **Cluster G — config resilience.** Bisect-on-NAK for CFG-VALSET and log the offending key by name; guard the VALGET frame with `cfg_batch_mutex_`; fix the VALGET completion heuristic.
6. **Cluster F/H — move reinit off the event thread; give NTRIP `curl` an abort/timeout so the destructor can't hang.**

> Suggest landing Cluster A + C + D first (these are the crash/wedge fixes), building + soak-testing on the F9R station, then E/G/F as a second pass.

## 4. Findings detail

## P0 findings (2)

### NODE-01 — rtcm_callback TOCTOU: lock-free dev_valid()/attached() check then blocking write_buffer races a concurrent close_devh() (use-after-free of devh_)

- **Severity:** P0  **Probability:** high  **Class:** TOCTOU / use-after-free / data-race  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:1543-1570`

**What's wrong.** rtcm_callback runs on a subscription executor thread (default callback group). It checks usbc_->dev_valid() (1543) and usbc_->attached() (1550) with no lock, then calls the blocking usbc_->write_buffer(...) (1570). The launch files run the node in component_container_mt (MultiThreadedExecutor), so this runs concurrently with handle_usb_events_callback (node 1283, callback_group_usb_events_timer_). Inside libusb_handle_events_timeout, libusb dispatches Connection::hotplug_detach_callback (usb.cpp 489) which calls close_devh() -> libusb_close(devh_); devh_=nullptr; attached_=false (usb.cpp 983-985). write_buffer takes write_mutex_ (usb.cpp 563) but close_devh() does NOT take write_mutex_, so there is no mutual exclusion between the close and the write. If detach lands between the check and the libusb_bulk_transfer, write_buffer dereferences a closed/dangling devh_ inside libusb -> use-after-free / undefined behavior. Even when the check passes, devh_ can be torn down mid-transfer.

**Evidence.** Check without lock: line 1543 `if (usbc_ == nullptr || !usbc_->dev_valid())`, line 1550 `if (!usbc_->attached())`; blocking write at 1570 `usbc_->write_buffer(data_out.data(), data_out.size());`. write_buffer locks write_mutex_ (usb.cpp 563) around libusb_bulk_transfer(devh_,...) (565-571). close_devh (usb.cpp 971-987) calls libusb_close(devh_) and sets devh_=nullptr WITHOUT acquiring write_mutex_. hotplug_detach_callback (usb.cpp 489-507) invokes close_devh() from the libusb event thread (handle_usb_events_callback, node 1283). dev_valid()/attached() read plain non-atomic members dev_/attached_ (usb.hpp 234-237, 287-290).

**Repro.** Run under component_container_mt with NTRIP feeding /ntrip_client/rtcm at a steady rate, then physically unplug (or USB-autosuspend) the receiver while RTCM is mid-write. The detach callback closes devh_ on the events thread while write_buffer is dereferencing it on the subscription thread. Expect intermittent crash / libusb assertion / UB under repeated hotplug.

**Suggested fix.** Serialize device teardown against writers: have close_devh() (and hotplug_detach_callback) acquire write_mutex_ before libusb_close/devh_=nullptr, and have write_buffer re-validate devh_!=nullptr && attached_ while holding write_mutex_ (move the validity check inside the lock). Make attached_/devh_ access consistent (single lock or atomics). Do not rely on the unlocked dev_valid()/attached() pre-check for safety.

**Verification.** CONFIRMED real against live source at /home/gtec/ublox_dgnss_audit (branch usb-stability, 0.7.4). All cited mechanics verified:

- rtcm_callback (ublox_dgnss_node.cpp:1541-1574): lock-free usbc_->dev_valid() at 1543, usbc_->attached() at 1550, then blocking usbc_->write_buffer(...) at 1570. Matches claim.
- write_buffer (usb.cpp:551-594): takes write_mutex_ (563) only around libusb_bulk_transfer(devh_,...) (565-571). It does NOT re-validate devh_/attached_ inside the lock.
- close_devh (usb.cpp:971-987): does libusb_close(devh_); devh_=nullptr; attached_=false (983-985) with NO write_mutex_ acquisition. Confirmed no mutual exclusion vs write_buffer's libusb_bulk_transfer.
- hotplug_detach_callback (usb.cpp:489-505) calls close_devh() at 498 from inside libusb event dispatch, which runs via handle_usb_events() (node 1283) on callback_group_usb_events_timer_ (Reentrant, node 134-136), 10ms wall timer (377-379).
- Members are plain non-atomic: devh_ (usb.hpp:111), dev_ (112), attached_ (145), write_mutex_ (153). No std::atomic. So dev_valid()/attached() reads race the detach-thread writ …

---

### USB-04 — rtcm_callback does a lock-free dev_valid()/attached() check then a blocking write_buffer — TOCTOU against hotplug detach + executor stall

- **Severity:** P0  **Probability:** high  **Class:** TOCTOU  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:1543-1573`

**What's wrong.** rtcm_callback validates usbc_->dev_valid() (1543) and usbc_->attached() (1550) with no lock, then falls through to a blocking synchronous write_buffer (1570 -> usb.cpp:551). Two problems: (1) TOCTOU — between the attached() check and the libusb_bulk_transfer dereference of devh_ (usb.cpp:565), the event thread can run hotplug_detach_callback -> close_devh and set devh_=nullptr (usb.cpp:984), so the bulk transfer is issued on a freed/null handle. The try/catch (node.cpp:1569-1573) only catches UsbException — a segfault or libusb assertion from a null/freed handle is not an exception. (2) write_buffer is synchronous with timeout_ms_=250 (usb.cpp:44,571); if the device is wedged (USB-01), every inbound RTCM message blocks the subscription thread for up to 250ms while holding write_mutex_, throttling RTCM and any other async write that needs write_mutex_.

**Evidence.** node.cpp:1543 `if (usbc_ == nullptr || !usbc_->dev_valid())`, 1550 `if (!usbc_->attached())`, 1570 `usbc_->write_buffer(...)` with the only guard being `catch (const usb::UsbException & e)` at 1571. write_buffer dereferences devh_ at usb.cpp:565 inside write_mutex_ but the null-check/handle-read is not atomic with close_devh's `devh_ = nullptr` (usb.cpp:984).

**Suggested fix.** Move the validity check inside the locked region in the Connection class (a write_buffer that internally locks, re-checks devh_!=nullptr && attached_, and returns/throws cleanly if not). Prefer the async write_buffer_async path used elsewhere so the subscription thread never blocks. Catch std::exception/all, and never dereference devh_ outside the state lock.

**Verification.** VERIFIED REAL as described. Read the cited code directly.

rtcm_callback (ublox_dgnss_node.cpp:1541-1574) does lock-free guards: dev_valid() at 1543 and attached() at 1550, then calls usbc_->write_buffer(...) at 1570 with only catch(const usb::UsbException&) at 1571. write_buffer (usb.cpp:551-594) reads devh_ at usb.cpp:566 inside write_mutex_ (lock at 563).

Concurrency is genuinely possible (the load-bearing question): the node is a component (RCLCPP_COMPONENTS_REGISTER_NODE, node.cpp:4381) and EVERY launch file uses executable 'component_container_mt' (MultiThreadedExecutor) — confirmed in ublox_dgnss/launch/*.py. The rtcm subscription is on callback_group_rtcm_timer_ (node.cpp:363) and the usb-events timer is on callback_group_usb_events_timer_ (node.cpp:379), both MutuallyExclusive but DIFFERENT groups, so they run concurrently on separate threads under the MT executor.

The detach path: handle_usb_events_callback (node.cpp:1255) -> usbc_->handle_usb_events() -> libusb_handle_events_timeout (usb.cpp:947) fires hotplug_detach_callback (usb.cpp:489) -> close_devh() (usb.cpp:498). …

---

## P1 findings (11)

### LIFE-02 — 10ms USB event timer can still fire and reap transfers concurrently with shutdown's cleanup_all_transfers / clear

- **Severity:** P1  **Probability:** high  **Class:** data-race  **Verify confidence:** medium
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:122-129 (on_shutdown lambda), 377-379 (10ms wall timer), 1255-1296 (handle_usb_events_callback)`

**What's wrong.** shutdown() / close_devh() / cleanup_all_transfers() are invoked from the on_shutdown context callback (node.cpp 122-129) and from the node destructor (451-457). Meanwhile handle_usb_events_callback runs every 10ms on callback_group_usb_events_timer_ and calls usbc_->handle_usb_events() -> libusb_handle_events_timeout (usb.cpp 947), which reaps transfers and runs callback_in/out, which themselves mutate transfer_queue_ (push via submit_transfer 795, delete user_data 702/640) and re-queue new transfers (706-714 / 644-652). The on_shutdown lambda only sets keep_running_=false and calls usbc_->shutdown(); it does NOT first cancel/stop handle_usb_events_timer_ nor wait for an in-flight handle_usb_events_callback to finish. So libusb_handle_events can be executing (touching transfer_queue_ and the transfer buffers) on the timer thread at the very moment cleanup_all_transfers() clears the deque and frees those same transfer_t objects on the shutdown thread. cleanup_all_transfers takes transfer_queue_mutex_, but the callbacks invoked from inside libusb_handle_events also free user_data and call exception_cb_fn_ outside that lock, and libusb_free_transfer in ~transfer_t races the libusb event handler. This is the concurrency half of LIFE-01.

**Evidence.** node.cpp 122-129: on_shutdown([this](){ keep_running_=false; if(usbc_!=nullptr){usbc_->shutdown();} }); — no timer cancel, no barrier. node.cpp 377: create_wall_timer(10ms, handle_usb_events_callback, callback_group_usb_events_timer_). handle_usb_events (usb.cpp 939-969) still calls libusb_handle_events_timeout even though keep_running_ guard at 941 only blocks the FIRST line — but shutdown() sets keep_running_=false BEFORE close_devh, yet a callback already past line 941 continues. close_devh frees transfers with no synchronization against that running event-handler.

**Suggested fix.** In on_shutdown and ~UbloxDGNSSNode, FIRST cancel handle_usb_events_timer_ and ensure no event callback is in flight (the MutuallyExclusive callback group plus an explicit drain), THEN call usbc_->shutdown(). Order must be: stop 10ms timer -> ensure current handle_usb_events_callback returned -> cancel transfers -> pump events to reap (LIFE-01) -> free buffers -> libusb_close. Consider a shutting_down_ atomic checked at the top of handle_usb_events_callback so a late timer tick is a no-op.

**Verification.** VERIFIED REAL, with one mechanism correction and a severity downgrade from P0 to P1.

Confirmed code facts:
- node.cpp:122-129 on_shutdown lambda only sets keep_running_=false and calls usbc_->shutdown(); it does NOT cancel handle_usb_events_timer_ nor drain an in-flight callback. ~UbloxDGNSSNode (451-457) is identical (no timer stop/barrier).
- node.cpp:377-379 handle_usb_events_timer_ is a 10ms wall timer on callback_group_usb_events_timer_ (MutuallyExclusive, created 134-135).
- node.cpp:1255-1296 handle_usb_events_callback checks `if(!keep_running_)` once at line 1257, then calls usbc_->handle_usb_events().
- usb.cpp:939-969 handle_usb_events checks keep_running_ at 941 once, then calls libusb_handle_events_timeout(947), which reaps transfers and runs callback_in/callback_out.
- shutdown() (989-1006) sets keep_running_=false then close_devh() -> cleanup_all_transfers() (859-891) which at 887 does transfer_queue_.clear().
- CRITICAL CONTEXT the report did not state but I confirmed: the node is loaded into component_container_mt (all launch files, e.g. ublox_dgnss/launch/ublox_rove …

---

### LIFE-03 — libusb_close called on a detached device with the documented 'hangs if detached already' comment, no guard

- **Severity:** P1  **Probability:** medium  **Class:** deadlock  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/usb.cpp:976-985`

**What's wrong.** close_devh() unconditionally releases interfaces, re-attaches kernel driver, and calls libusb_close(devh_) at line 983, which carries the in-code comment 'hangs if the device has been detached already'. On the F9R shutdown path close_devh is reached both from hotplug_detach_callback (498) and from shutdown() (1005). If the physical device was already removed (USB unplug, or a NO_DEVICE streak), libusb_release_interface / libusb_attach_kernel_driver / libusb_close can block. Because shutdown() runs on the rclcpp shutdown thread, a hang here wedges process exit. There is no check of attached_ / a NO_DEVICE state before close, and libusb_release_interface is hard-coded to loop if_num 0..1 (line 977) regardless of num_interfaces_ (X20P single-interface devices release a non-claimed interface 1).

**Evidence.** usb.cpp 977-983: for(int if_num=0; if_num<2; if_num++){ libusb_release_interface(devh_,if_num); if rc>=0 libusb_attach_kernel_driver(devh_,if_num);} libusb_close(devh_); // hangs if the device has been detached already. Hard-coded <2 ignores num_interfaces_ (set at open_device 298).

**Suggested fix.** Guard close with the known device state: if a detach was already observed (driver_state_==DISCONNECTED or a NO_DEVICE streak), skip release/attach and call libusb_close only when safe, or skip close entirely on detach (libusb_exit will reclaim). Loop if_num over num_interfaces_ not a hard-coded 2. Optionally run close on a separate thread with a timeout so a hang cannot wedge shutdown.

**Verification.** Verified against live source at /home/gtec/ublox_dgnss_audit/ublox_dgnss_node/src/usb.cpp:971-987. All cited specifics confirmed: close_devh() at 976-986 loops `for(int if_num=0; if_num<2; if_num++)` (hard-coded 2), calls libusb_release_interface(devh_,if_num), then libusb_attach_kernel_driver only if rc>=0, then `libusb_close(devh_); // hangs if the device has been detached already` (exact comment at line 983). No guard on attached_ / driver_state_ / no_device_streak_ before the release/attach/close sequence. close_devh() is reached from hotplug_detach_callback (line 498) and from shutdown() (line 1005), and shutdown() runs from ~Connection() (line 1010). So an already-removed device hitting the shutdown/destructor path can block in libusb_close per the developer's own documented hazard, wedging process exit — the claimed deadlock mechanism is real and the file:line is accurate.

Secondary claim also confirmed: open uses num_interfaces_ correctly (claim loop at usb.cpp:344-355; num_interfaces_ set at 298, declared usb.hpp:134, can be 1 for X20P 0x050c/0x050d), but close hard-codes ` …

---

### LIFE-07 — rtcm_callback checks dev_valid()/attached() without a lock then calls blocking write_buffer; races detach/close freeing devh_

- **Severity:** P1  **Probability:** medium  **Class:** TOCTOU  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:1541-1574`

**What's wrong.** rtcm_callback runs on the RTCM subscription callback group and tests usbc_->dev_valid() (1543) and usbc_->attached() (1550) without any lock, then calls usbc_->write_buffer (1570), which performs a blocking libusb_bulk_transfer on devh_ (usb.cpp 565). Between the attached() check and the write, hotplug_detach_callback (running from libusb_handle_events on the 10ms timer thread) can call close_devh() and set devh_=nullptr / attached_=false (usb.cpp 984-985), or shutdown() can free the handle. write_buffer dereferences devh_ with no validity recheck under write_mutex_; write_mutex_ only serializes concurrent writes, it does not protect against devh_ being closed underneath. On shutdown this is another path that can submit a bulk transfer to a handle being torn down, contributing to the heap/handle instability.

**Evidence.** node.cpp 1543 if(usbc_==nullptr || !usbc_->dev_valid()) ... 1550 if(!usbc_->attached()) ... 1570 usbc_->write_buffer(...). usb.cpp 562-571 write_buffer takes only write_mutex_ then libusb_bulk_transfer(devh_,...) with no devh_!=nullptr recheck. close_devh (984) sets devh_=nullptr from the event-timer thread with no shared lock against rtcm_callback.

**Suggested fix.** Guard devh_ lifetime with the same mutex used in write_buffer: re-check devh_!=nullptr && attached_ inside the write_mutex_ critical section before libusb_bulk_transfer, and hold that mutex across close_devh's devh_ teardown so a write cannot run against a half-closed handle. Set keep_running_/attached_ atomically and re-test immediately before submit.

**Verification.** Confirmed by reading the live source. rtcm_callback (node.cpp 1543, 1550) calls usbc_->dev_valid() and usbc_->attached() with no lock, then write_buffer (1570). write_buffer (usb.cpp 551-594) takes only write_mutex_ (563) and calls libusb_bulk_transfer(devh_,...) at 565-566 with no devh_!=nullptr / attached_ recheck inside the lock. hotplug_detach_callback (usb.cpp 489-505), invoked from libusb_handle_events on the 10ms wall timer, calls close_devh() (498) which does libusb_close(devh_) then devh_=nullptr; attached_=false (983-985). close_devh does NOT acquire write_mutex_, so it can run concurrently with / between the attached() check and the bulk transfer. The TOCTOU race and unsynchronized devh_ teardown vs write_buffer are real exactly as described; cited file:lines are accurate.

Two corrections that do not change the verdict: (1) dev_valid() actually tests dev_!=nullptr (usb.hpp 236), not devh_ — the teardown gate the finding depends on is attached()/attached_, which is correct. (2) Passing a closed/freed libusb_device_handle to libusb_bulk_transfer is undefined behavior (use-a …

---

### NODE-03 — Hotplug attach/detach callbacks and full re-initialization run inline on the USB-events timer thread, stalling the libusb event pump

- **Severity:** P1  **Probability:** high  **Class:** liveness / re-entrancy  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:1283, 1727-1750`

**What's wrong.** libusb dispatches hotplug callbacks synchronously from within libusb_handle_events_timeout (usb.cpp 947), which is called by handle_usb_events_callback (node 1283) on callback_group_usb_events_timer_ (10ms wall timer). On attach, Connection::hotplug_attach_callback (usb.cpp 468) calls hp_attach_cb_fn_() -> node hotplug_attach_callback (1727), which for a reconnection calls perform_usb_initialization() (1740). perform_usb_initialization does the full blocking usbc_->init() + init_async() + ublox_dgnss_init_async() (462-489) INLINE — all executed while still inside libusb_handle_events_timeout on the event-pump thread. That blocks the only thread that services USB IN/OUT completion events for the entire (potentially multi-hundred-ms) init sequence, starving in/out transfer completion. Worse, on init failure perform_usb_initialization calls usbc_->shutdown() and rclcpp::shutdown() (499-502) from inside the libusb event callback, reentrantly tearing down the very connection whose event loop is on the stack.

**Evidence.** handle_usb_events_callback -> usbc_->handle_usb_events() (node 1283) -> libusb_handle_events_timeout (usb.cpp 947) -> hotplug_attach_callback (usb.cpp 468) -> (hp_attach_cb_fn_)() (usb.cpp 482) -> node hotplug_attach_callback (1727) -> perform_usb_initialization() (1740) which calls usbc_->init() (472), init_async() (482) and on error usbc_->shutdown()/usbc_.reset()/rclcpp::shutdown() (499-502). Same thread also where hotplug_detach_callback -> close_devh()/libusb_close runs (usb.cpp 489-507, 983).

**Repro.** Hotplug-cycle the device; observe that during reconnection the 10ms USB events timer overruns, IN/OUT completions are delayed, and any init exception triggers shutdown() from inside the event callback. Rapid plug/unplug can reenter init while a prior transfer teardown is pending.

**Suggested fix.** Do not perform blocking init or shutdown inside the hotplug callback / event-pump thread. Set a flag/post to a separate timer or thread to perform (re)initialization and shutdown outside libusb_handle_events. Keep hotplug callbacks minimal (set state + notify).

**Verification.** Verified the entire call chain against live source. Confirmed: handle_usb_events_timer_ is a 10ms wall timer on callback_group_usb_events_timer_ (MutuallyExclusive) at node.cpp:377-379; handle_usb_events_callback -> usbc_->handle_usb_events() at node.cpp:1283; handle_usb_events() -> libusb_handle_events_timeout(ctx_) at usb.cpp:947; Connection::hotplug_attach_callback at usb.cpp:468 calls open_device() (478) then (hp_attach_cb_fn_)() at usb.cpp:482 SYNCHRONOUSLY on the stack inside libusb_handle_events_timeout; hp_attach_cb_fn_ is std::bind to node hotplug_attach_callback at node.cpp:1727, which on reconnection (has_been_connected_before_) calls perform_usb_initialization() at node.cpp:1740; perform_usb_initialization (node.cpp:462-525) runs usbc_->init() (472), init_async() (482), ublox_dgnss_init_async() (487) inline, and on any error path calls usbc_->shutdown(), usbc_.reset(), rclcpp::shutdown() (498-523). All file:line citations in the finding are accurate.

The core mechanism is REAL: this work executes on the stack of libusb_handle_events_timeout, the single thread (MutuallyEx …

---

### NODE-04 — Driver-state and attached_ flags read across threads as plain non-atomic members; checks in callbacks race writers and can be stale/torn

- **Severity:** P1  **Probability:** high  **Class:** data-race  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/include/ublox_dgnss_node/usb.hpp:145-147, 287-295`

**What's wrong.** attached_ (bool, usb.hpp 145), devh_/dev_ (pointers, 111-112) and driver_state_ (enum, 147) are plain members with no atomicity or memory ordering. They are written on the USB-events thread (hotplug_attach/detach, handle_usb_events no_device_streak path setting attached_=false at usb.cpp 954) and read on subscription/timer threads (rtcm_callback 1543/1550, usb_ready_for_rxm_input 1490, handle_usb_events_callback state checks 1269/1276, rtcm_timer_callback 2031, send_parameter_to_device 1338). Concurrent read/write of these without synchronization is a data race (UB), and even ignoring UB the values are checked-then-used with a wide TOCTOU window. The no_device_streak_ heuristic (usb.cpp 953) flips attached_ to false from the events thread with no fence, so the subscription thread may observe a stale 'attached' for an unbounded time.

**Evidence.** Declarations usb.hpp 145 `bool attached_;`, 147 `USBDriverState driver_state_;`, accessors return raw values (287-295). Writers: hotplug_attach_callback attached_=true (usb.cpp 481), hotplug_detach_callback/close_devh attached_=false (usb.cpp 501,985), handle_usb_events attached_=false on no_device streak (usb.cpp 954), driver_state_ transitions (usb.cpp 479,499). Readers on other threads: node 1543,1550,1490,1269,1276,1338,2031.

**Repro.** Under MT executor with hotplug events, observe inconsistent behavior: writers proceed after attached_ already cleared, or callbacks keep believing CONNECTED after a no-device streak; ThreadSanitizer flags races on attached_/devh_/driver_state_.

**Suggested fix.** Make attached_/driver_state_ std::atomic (or guard all access with a single state mutex), and pair the device-validity check with the actual use under one lock (see NODE-01). At minimum publish state changes with release/acquire semantics.

**Verification.** VERIFIED as described. Confirmed all cited file:line.

Declarations (usb.hpp): `bool attached_;` (145), `USBDriverState driver_state_;` (147), `libusb_device_handle * devh_;` (111), `libusb_device * dev_;` (112) — all plain non-atomic members. Accessors are lock-free: attached() returns raw attached_ (287-290), driver_state() returns raw driver_state_ (292-295), dev_valid() returns dev_ != nullptr (234-237) — no fence/mutex.

Writers (usb.cpp, on USB events/hotplug thread): hotplug_attach driver_state_=CONNECTED (479), attached_=true (481); hotplug_detach driver_state_=DISCONNECTED (499), attached_=false (501); handle_usb_events no_device streak attached_=false (954); close_devh devh_=nullptr + attached_=false (984-985). Confirmed no_device_streak_ flips attached_ false from the events thread with no fence (952-955), matching the claim.

Readers (node.cpp, on other callback groups): rtcm_callback dev_valid() (1543) and attached() (1550) then blocking write_buffer (1570); usb_ready_for_rxm_input dev_valid()/attached() (1490); handle_usb_events_callback driver_state() (1269/1276); send …

---

### UBX-01 — NMEA branch writes one byte past USB transfer buffer (buf[len]=0)

- **Severity:** P1  **Probability:** medium  **Class:** buffer-overflow  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:1619-1628`

**What's wrong.** In ublox_in_callback, when the first byte is '$' (0x24) the code treats the transfer as NMEA and executes `buf[len] = 0;` to NUL-terminate before logging with %s. `buf` is `transfer_in->buffer`, a libusb bulk-transfer buffer of exactly IN_BUFFER_SIZE = 64*200 = 12800 bytes (usb.hpp:48, usb.cpp:757). `len` is `transfer_in->actual_length`, which can be up to the full buffer length (12800). When a maximally-sized transfer begins with 0x24, `buf[12800]=0` writes one byte beyond the 12800-byte heap allocation — a 1-byte heap overflow. The data is fully attacker/device-controlled (any device or USB-fuzz that sends a 12800-byte bulk-in beginning with '$'). Even for smaller len it writes at buf[len] which is past the actual data but normally still inside the 12800 buffer; the overflow only manifests at len==IN_BUFFER_SIZE, but that is reachable because make_transfer_in sizes the buffer to exactly IN_BUFFER_SIZE.

**Evidence.** Line 1622 `buf[len] = 0;` with buf=transfer_in->buffer (1587) and len=transfer_in->actual_length (1586). Buffer is `transfer->buffer->resize(IN_BUFFER_SIZE)` (usb.cpp:757) and libusb_fill_bulk_transfer uses buffer->size() (usb.cpp:766-767). IN_BUFFER_SIZE is `64 * 200` (usb.hpp:48).

**Repro.** Feed a 12800-byte bulk-in transfer whose first byte is 0x24; ASAN flags heap-buffer-overflow WRITE at the buf[len]=0 line.

**Suggested fix.** Do not write at buf[len]. Either bound-check (`if (len < IN_BUFFER_SIZE) buf[len]=0;`) or, better, build a std::string from (buf, len) and log that, never mutating the transfer buffer. Also reserve one extra byte in IN_BUFFER_SIZE if NUL-termination is required.

**Verification.** Verified against live source. ublox_dgnss_node.cpp:1586-1587 set len=transfer_in->actual_length and buf=transfer_in->buffer; line 1621 checks buf[0]==0x24 (NMEA '$') and line 1622 executes buf[len]=0 unconditionally. buf points at the libusb async transfer buffer, which is a std::vector<u_char> (usb.hpp:53,67) resized to exactly IN_BUFFER_SIZE=64*200=12800 (usb.hpp:48) in make_transfer_in (usb.cpp:757) and passed to libusb_fill_bulk_transfer with buffer->size()=12800 (usb.cpp:767). libusb sets actual_length up to the full requested length; a device delivering a full 12800-byte bulk-in with no short packet yields actual_length==12800, so buf[12800]=0 is a 1-byte out-of-bounds heap write past the vector's allocation. Data is fully device-controlled, so the leading-'$' precondition is trivially satisfiable. This is a genuine heap buffer overflow exactly as described at the cited file:line.

Caveats affecting severity (kept at P1, not P0): the OOB write is a single 0x00 byte at the one-past-end position, not arbitrary-length corruption, and only triggers at the exact boundary len==IN_BUF …

---

### UBX-02 — from_buf_build trusts wire data and over-reads on short frames

- **Severity:** P1  **Probability:** medium  **Class:** buffer-over-read  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx.hpp:63-73`

**What's wrong.** Frame::from_buf_build() unconditionally reads buf[2], buf[3], `*reinterpret_cast<u2_t*>(&buf[4])` (length), sets payload=&buf[6], and reads buf[buf.size()-2] and buf[buf.size()-1] for checksums. The only guard at the call site (ublox_in_callback, node.cpp:1631) is `len > 2 && buf[0]==sync1 && buf[1]==sync2`. A frame with len==3,4,5 passes that guard, but from_buf_build then reads buf[4]/buf[5] (length field) out of bounds, and `payload=&buf[6]` points past the vector. The length field is read but NEVER validated against buf.size(): there is no check that buf.size() == 6 + length + 2. A device (or fuzzed USB) that reports actual_length=4 with a bogus length field causes an out-of-bounds read of the std::vector backing store.

**Evidence.** ubx.hpp:69 `length = *reinterpret_cast<u2_t *>(&buf[4]);` and :70 `payload = reinterpret_cast<ch_t *>(&buf[6]);` with no size check; caller guard only `len > 2` at node.cpp:1631. There is no `buf.size() >= 8 && buf.size() == 6+length+2` validation anywhere before use.

**Repro.** Send a bulk-in of 0x62 0xB5? no — send 0xB5 0x62 0x01 (len=3); guard len>2 passes, from_buf_build reads buf[4],buf[5] OOB.

**Suggested fix.** In from_buf_build (or before calling it) require buf.size() >= 8 and buf.size() == 6 + (buf[4] | buf[5]<<8) + 2; reject/drop frames that fail. Treat the length field as untrusted.

**Verification.** VERIFIED REAL at the exact cited lines. ubx.hpp:63-73 from_buf_build(): line 69 `length = *reinterpret_cast<u2_t*>(&buf[4])` reads buf[4]/buf[5]; line 70 `payload = reinterpret_cast<ch_t*>(&buf[6])`; lines 71-72 read buf[size-2]/buf[size-1]. Caller ublox_in_callback (node.cpp:1631) guards only `len > 2 && buf[0]==SYNC1 && buf[1]==SYNC2`, then resizes the vector to exactly `len` (node.cpp:1633-1635, memcpy of `len` bytes) and calls from_buf_build (1636). len = transfer_in->actual_length straight from libusb (node.cpp:1586) with NO minimum-length enforcement, so len can be 3/4/5.

Two confirmed defects, both genuine:
1) Short-frame heap over-read: for len==3 or 4, reading buf[4]/buf[5] at ubx.hpp:69 is an out-of-bounds read of the std::vector backing store; for len==5, buf[5] is OOB. &buf[6] at line 70 is OOB pointer formation when size<6 (and one-past-end at size==6).
2) length field never validated: confirmed there is NO `buf.size() == 6 + length + 2` (or `>= 8`) check anywhere before use. Downstream consumers use payload+length unchecked: e.g. node.cpp:2485-2486 / 2493-2495 construc …

---

### UBX-08 — ubx_check_sum/build_frame_buf iterate `length` payload bytes without validating against buf size

- **Severity:** P1  **Probability:** medium  **Class:** buffer-over-read  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx.hpp:45-87`

**What's wrong.** ubx_check_sum() calls build_frame_buf(), which loops `for (int i=0;i<length;i++) buf.push_back(payload[i]);`. After from_buf_build, `payload` points at &buf[6] of the ORIGINAL vector and `length` is the untrusted wire length field. If length > actual payload bytes present (UBX-02), this reads `length` bytes starting at original buf[6], over-reading the heap, and also rebuilds a frame of the wrong size. Because every incoming frame is checksum-checked via ubx_frame_checksum_check → ubx_check_sum, this over-read happens for essentially every malformed/short frame before it can be rejected. Also `length` is `u2_t` but the loop counter `int i` — fine for <=65535, but the trust in `length` is the core defect.

**Evidence.** ubx.hpp:54 `for (int i = 0; i < length; i++) buf.push_back(payload[i]);` where length is wire-controlled (set at ubx.hpp:69) and payload=&buf[6] (70); ubx_check_sum at 75-87 invokes build_frame_buf at 77; checksum is run on every inbound frame (node.cpp:2110, 2094).

**Repro.** Inbound frame with length field larger than received bytes triggers OOB read inside ubx_check_sum during the validity check itself.

**Suggested fix.** Validate the frame size against the length field before any checksum/rebuild (reject if original buf.size() != 6+length+2). Compute checksum directly over the received bytes rather than rebuilding from a trusted-length payload pointer.

**Verification.** VERIFIED real. Read ubx.hpp:45-87 and node.cpp:1619-1700, 2090-2160.

Mechanism confirmed: node.cpp:1632-1636 builds the inbound frame as buf sized to the ACTUAL usb transfer length `len` (frame->buf.resize(len); memcpy from received bytes), then calls from_buf_build(). from_buf_build() (ubx.hpp:63-73) sets length = *reinterpret_cast<u2_t*>(&buf[4]) — the untrusted 16-bit wire length field (u2_t=uint16_t per ubx_types.hpp:27, so up to 65535) — and payload = &buf[6]. NO validation that buf.size() == 6+length+2 exists anywhere between from_buf_build and use.

ubx_frame_checksum_check (node.cpp:2090) is the FIRST thing done with the frame (called at 2110 for inbound, 2157 for outbound) and calls ubx_check_sum() -> build_frame_buf() (ubx.hpp:54-56) which loops `for (int i=0;i<length;i++) buf.push_back(payload[i])`. When the wire `length` exceeds the actual payload bytes present (len-8), this reads past the original buffer = heap over-read. payload_to_hex() (node.cpp:2108/2155, ubx.hpp:102) does the same over-read for logging.

CORRECTION / escalation of mechanism: build_frame_buf first d …

---

### USB-01 — Persistent LIBUSB_ERROR_TIMEOUT on an attached device has NO recovery path — the multi-hour wedge

- **Severity:** P1  **Probability:** high  **Class:** wedge-recovery  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/usb.cpp:655-715, 947-969`

**What's wrong.** The active data path is fully asynchronous (read_chars at usb.cpp:507 is only used in commented-out code in node.cpp:1952). The device is kept alive by a self-perpetuating chain of IN transfers: each callback_in resubmits the next IN transfer (usb.cpp:706-714) and callback_out does the same (usb.cpp:644-652). If the device stops delivering data but remains enumerated (no DEVICE_LEFT hotplug, no NO_DEVICE error) — e.g. firmware hang, endpoint stall cleared to nothing, USB babble, link suspend — the IN transfer completes with status LIBUSB_TRANSFER_TIMED_OUT (filled with timeout 0 at usb.cpp:768 means the per-transfer libusb timeout is infinite, but a STALL/ERROR/short-read with SHORT_NOT_OK can also trip the error branch). On any non-COMPLETED, non-NO_DEVICE status, callback_in goes to the else branch, logs via err_count_ (suppressed after 10, usb.cpp:689), marks completed and deletes the sp, then STILL resubmits a fresh IN transfer (706-714). There is NO clear_halt, NO libusb_reset_device (grep confirms libusb_reset_device appears nowhere), NO reopen, NO endpoint stall recovery. handle_usb_events (947) only special-cases LIBUSB_ERROR_NO_DEVICE (952) and LIBUSB_ERROR_INTERRUPTED (949); a libusb_handle_events that keeps returning 0 with transfers silently timing out is invisible to it. There is no watchdog tracking 'no IN transfer COMPLETED for N seconds'. Result: the node spins its 10ms event timer forever, publishes nothing, and never self-heals — the reported multi-hour wedge.

**Evidence.** usb.cpp:657 `if (transfer->status == ... LIBUSB_TRANSFER_COMPLETED)` else-branch 660-692 handles TIMED_OUT/STALL/ERROR by only incrementing err_count_; 706 `if (attached_ && queued_transfer_in_num() == 0)` resubmits unconditionally with no halt-clear/reset. handle_usb_events 952 `case LIBUSB_ERROR_NO_DEVICE` is the ONLY error escalation; TIMEOUT never reaches here. No libusb_clear_halt/libusb_reset_device anywhere in usb.cpp. timeout_ms_=250 (usb.cpp:44) only applies to the unused sync read_chars; async transfers are filled with timeout 0 (usb.cpp:768) = never time out at libusb level, so a stalled IN endpoint just sits pending forever and queued_transfer_in_num() stays 1, so even the resubmit guard at 706 prevents re-arming.

**Suggested fix.** Add a stall/timeout recovery state machine: (a) give async IN transfers a finite timeout in libusb_fill_bulk_transfer (usb.cpp:768) so STALL/timeout is observable; (b) on LIBUSB_TRANSFER_STALL call libusb_clear_halt(devh_, ep_data_in_addr_) before resubmit; (c) add a watchdog timestamp updated on every LIBUSB_TRANSFER_COMPLETED in callback_in, checked by handle_usb_events_callback; if no completion for N seconds while attached_, escalate to a controlled close_devh()+reopen (or libusb_reset_device) instead of blindly resubmitting; (d) bound consecutive non-COMPLETED IN statuses and force reconnect rather than only muting logs after 10.

**Verification.** Verified against live source at /home/gtec/ublox_dgnss_audit. All load-bearing claims confirmed:

- callback_in (usb.cpp:655-715): COMPLETED branch resets err_count_=0; else-branch (660-692) handles ERROR/TIMED_OUT/STALL/OVERFLOW/default by only building a message and calling exception_cb_fn_ gated by ++err_count_<10 (line 689). NO clear_halt, NO reset, NO reopen. NO_DEVICE returns early (675-679).
- Unconditional resubmit at 706-714: fresh IN transfer queued if attached_ && queued_transfer_in_num()==0, with zero stall/halt recovery in between. Confirmed.
- grep: libusb_reset_device and libusb_clear_halt appear NOWHERE in src/ or include/. Confirmed.
- handle_usb_events (947-960): libusb_handle_events_timeout return code only special-cases LIBUSB_ERROR_INTERRUPTED (949) and LIBUSB_ERROR_NO_DEVICE (952, escalates to attached_=false after kNoDeviceThreshold=3). Per-transfer STALL/TIMED_OUT statuses never surface here since handle_events returns 0 on success. Confirmed.
- Async transfers filled with timeout 0: make_transfer_in (usb.cpp:768) and make_transfer_out (747) pass 0 as the time …

---

### USB-03 — devh_ and attached_ are read/written across three threads with no synchronization (data race; use-after-free on devh_)

- **Severity:** P1  **Probability:** high  **Class:** data-race  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/usb.cpp:477-505, 514-515, 535, 565, 644, 706, 779, 944-985`

**What's wrong.** devh_ (libusb_device_handle*) and attached_ (plain bool, not atomic — usb.hpp:111,145) are shared mutable state touched by at least three threads with NO lock protecting them. (1) The libusb event thread = the 10ms usb-events timer in callback_group_usb_events_timer_ (node.cpp:377-379) which calls handle_usb_events -> libusb_handle_events_timeout, which invokes the hotplug callbacks and the transfer completion callbacks IN THAT THREAD: hotplug_detach_callback writes devh_=nullptr and attached_=false (usb.cpp:498->close_devh:984-985, 497-501); callback_in/out read attached_ (644,706) and read devh_ implicitly via resubmit; submit_transfer reads attached_/keep_running_ (779). (2) The subscription thread: rtcm_callback (node.cpp:1541) and ubx_esf_meas/rxm callbacks run in the node's DEFAULT callback group (subscriptions created with sub_options that set NO callback_group, node.cpp:386-409), which under the standard component_container_mt MultiThreadedExecutor runs on a different thread than the usb-events timer group. These call write_buffer (usb.cpp:551, reads devh_ at 565) and write_buffer_async->make_transfer_out (usb.cpp:744 reads devh_) concurrently. (3) The param/ubx timer threads (separate MutuallyExclusive groups, node.cpp:357,384) also issue async writes. write_mutex_ only serializes the libusb_bulk_transfer call itself (usb.cpp:534,563); it does NOT cover the devh_ pointer read against a concurrent devh_=nullptr in close_devh. A detach on the event thread can null devh_ at usb.cpp:984 between the rtcm thread's dev_valid()/attached() check (node.cpp:1543,1550) and its libusb_bulk_transfer(devh_,...) at usb.cpp:565 — classic TOCTOU producing a use-after-free / NULL-handle call into libusb.

**Evidence.** attached_ declared `bool attached_;` (usb.hpp:145) and devh_ `libusb_device_handle * devh_;` (usb.hpp:111) — neither atomic, no mutex guarding them. close_devh writes `devh_ = nullptr; attached_ = false;` (usb.cpp:984-985) on the event-callback thread. rtcm_callback reads `usbc_->dev_valid()` and `usbc_->attached()` WITHOUT any lock (node.cpp:1543,1550) then calls write_buffer (node.cpp:1570) which dereferences devh_ at usb.cpp:565. Subscriptions get no callback group (node.cpp:400,408 pass only sub_options), so they are in a different executor group from callback_group_usb_events_timer_.

**Suggested fix.** Introduce a single mutex (or shared_mutex) that guards devh_/dev_/attached_/driver_state_ as a unit. dev_valid()/attached() plus the subsequent libusb_bulk_transfer must be performed while holding it (or copy devh_ under the lock and re-check non-null). Make attached_/keep_running_ std::atomic<bool> at minimum. Hold the same lock in close_devh while nulling devh_ and in the hotplug detach path. Assign the input subscriptions to a dedicated callback group and document/serialize against the event thread.

**Verification.** CONFIRMED. All cited file:lines verified against live source. devh_ (usb.hpp:111) and attached_ (usb.hpp:145) are plain non-atomic members; accessors dev_valid/devh_valid/attached (usb.hpp:234-241,287-290) are unlocked reads. close_devh() writes devh_=nullptr; attached_=false (usb.cpp:984-985) from hotplug_detach_callback (usb.cpp:498) which executes inside libusb_handle_events_timeout (usb.cpp:947) on the usb-events MutuallyExclusive timer group (node.cpp:134-135,377-379). handle_usb_events_callback also writes attached_=false at usb.cpp:954. Reader path: rtcm_callback (node.cpp:1543,1550) checks dev_valid()/attached() UNLOCKED, then write_buffer reads devh_ at usb.cpp:565; make_transfer_out reads devh_ at usb.cpp:744 with NO lock; submit_transfer reads attached_/keep_running_ at usb.cpp:779 unlocked; callback_in/out read attached_ at usb.cpp:644,706.

Threading model CONFIRMED as the decisive point: subscriptions are created with only sub_options and NO callback group (node.cpp:400,408), placing them in the node default group; ALL launch files use component_container_mt (=MultiThre …

---

### USB-05 — libusb_close on a detached device can hang the entire event thread (acknowledged in code comment)

- **Severity:** P1  **Probability:** medium  **Class:** deadlock  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/usb.cpp:971-987`

**What's wrong.** close_devh runs cleanup_all_transfers (which libusb_cancel_transfer's pending transfers, 859-891) then iterates if_num 0..1 calling libusb_release_interface + libusb_attach_kernel_driver (977-981), then libusb_close (983). The inline comment at 983 states 'hangs if the device has been detached already'. close_devh is invoked from hotplug_detach_callback (498) which is dispatched from inside libusb_handle_events on the usb-events timer thread (node.cpp:377). If libusb_close blocks, the 10ms event timer never returns: no further events are processed, no transfers complete, no other detach/attach is handled, and shutdown (989) which also calls close_devh (1005) will hang too. Additionally the interface loop hardcodes `if_num < 2` (977) regardless of num_interfaces_ (which is 1 for X20P vendor-specific per usb.cpp:316), so release/attach is attempted on a non-existent interface 1 for single-interface devices — return-code-guarded for attach (979) but libusb_release_interface(1) on a 1-interface device returns NOT_FOUND and is silently ignored, harmless but indicates the count is not respected.

**Evidence.** usb.cpp:983 `libusb_close(devh_);  // hangs if the device has been detached already`. close_devh called from hotplug_detach_callback usb.cpp:498, which is invoked synchronously within libusb_handle_events_timeout (usb.cpp:947) on the 10ms timer thread (node.cpp:377-379). Hardcoded loop `for (int if_num = 0; if_num < 2; if_num++)` at usb.cpp:977 ignores num_interfaces_.

**Suggested fix.** Do not call the full close (release_interface/attach_kernel_driver/libusb_close) from inside the libusb event callback thread for a device that libusb already reports gone. On DEVICE_LEFT, only mark state and free transfers; defer/skip libusb_close when the device is already detached (libusb auto-invalidates the handle), or perform close on a separate thread with a timeout/guard so the event loop cannot wedge. Bound the interface loop by num_interfaces_ instead of the literal 2.

**Verification.** Verified against live source. close_devh() at usb.cpp:971-987 is exactly as described: cleanup_transfer_queue()/cleanup_all_transfers() then a hardcoded `for (int if_num = 0; if_num < 2; if_num++)` (977) doing libusb_release_interface + libusb_attach_kernel_driver (978-981), then `libusb_close(devh_); // hangs if the device has been detached already` (983 — literal comment confirmed). close_devh is called from hotplug_detach_callback (usb.cpp:498) inside `if (attached_)`. libusb hotplug callbacks are dispatched synchronously from within libusb_handle_events_timeout (usb.cpp:947), which is driven by handle_usb_events() on the 10ms ROS wall timer (node.cpp:377-379, callback handle_usb_events_callback at 1255/1283). So a hang in libusb_close does block the event-pump thread and prevents all further event processing. shutdown() (1005) also calls close_devh(), so shutdown can hang too — both confirmed. The hardcoded `2` not respecting num_interfaces_ is also real: init() uses num_interfaces_ at usb.cpp:344/355, but close_devh uses literal 2; for vendor-specific X20P num_interfaces_==1 (us …

---

## P2 findings (13)

### LIFE-05 — NTRIP destructor join() can hang forever: streaming thread blocked in curl_easy_perform with no abort

- **Severity:** P2  **Probability:** medium  **Class:** deadlock  **Verify confidence:** high
- **Location:** `ntrip_client_node/src/ntrip_client_node.cpp:296-360 (DoStreaming), 363-374 (~NTRIPClientNode)`

**What's wrong.** ~NTRIPClientNode sets streaming_exit_.store(true) then streamingThread_.join() (366-369). streaming_exit_ is only checked at the top of the while loop (298); the thread spends essentially all its time blocked inside curl_easy_perform (302) streaming RTCM from the caster. There is no CURLOPT_TIMEOUT, CURLOPT_LOW_SPEED_LIMIT/TIME, or progress/xferinfo callback that returns non-zero when streaming_exit_ is set, so nothing breaks a live perform. If the caster keeps the connection open (the normal case), curl_easy_perform never returns, the while-condition is never re-evaluated, and join() blocks indefinitely — node/process shutdown hangs.

**Evidence.** ntrip 298 while(!streaming_exit_.load()){ ... 302 CURLcode res = curl_easy_perform(curlHandle_->handle); ...}. ~NTRIPClientNode 366-369: streaming_exit_.store(true); streamingThread_.join();. No CURLOPT_TIMEOUT / CURLOPT_XFERINFOFUNCTION wired in ApplyCurlOptions (224-241) or the static options (105-117) to observe streaming_exit_.

**Suggested fix.** Install a CURLOPT_XFERINFOFUNCTION (with CURLOPT_NOPROGRESS=0) that returns non-zero when streaming_exit_.load() is true, causing curl_easy_perform to abort promptly; or set CURLOPT_LOW_SPEED_TIME/LIMIT and a reasonable CURLOPT_TIMEOUT. Then join() will complete.

**Verification.** Verified against ntrip_client_node/src/ntrip_client_node.cpp. The code facts in the finding are correct: ~NTRIPClientNode (364-374) does streaming_exit_.store(true) then streamingThread_.join() (366-369); streaming_exit_ is only checked at the while-loop top (298); curl_easy_perform is at 302; and no CURLOPT_TIMEOUT / CURLOPT_LOW_SPEED_* / CURLOPT_XFERINFOFUNCTION / CURLOPT_NOPROGRESS exists anywhere (confirmed by grep: static opts 105-113 and ApplyCurlOptions 224-241 set none). So there is no in-flight abort path tied to streaming_exit_, and a blocked perform makes join() hang.

CORRECTION to the mechanism/severity: the finding's claim that perform "never returns" so join hangs "if the caster keeps the connection open (the normal case)" is wrong. WriteCallback (243-293) deliberately forces curl_easy_perform to return every 10 received RTCM records by returning size*nmemb-1 (lines 274-288, 281), which signals libcurl to abort the transfer (perform returns CURLE_WRITE_ERROR). This is the documented design (comment lines 98-101: "force it to exit in WriteCallback ... such that the ros2 …

---

### NODE-02 — 250ms blocking write_buffer on the subscription thread is a liveness hazard; rtcm sub uses the default callback group so it serializes other default-group callbacks

- **Severity:** P2  **Probability:** high  **Class:** deadlock / liveness (head-of-line blocking)  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:403-408, 1570`

**What's wrong.** rtcm_callback performs a synchronous libusb_bulk_transfer with timeout_ms_ (default 250ms per usb.cpp write_buffer) directly on the executor thread servicing the RTCM subscription. The RTCM subscription is created WITHOUT a dedicated callback group (1405-408 pass only sub_options, no callback group), so it lives in the node default mutually-exclusive group together with every other ungrouped subscription (all the input subs at 393-... and any default-group entities). When the device stalls/half-detaches, the bulk transfer blocks up to the full timeout; if write returns short/error it throws and the next RTCM message blocks again. During that window no other default-group callback can run on that executor thread, and a backlog of RTCM messages (QoS depth 10) compounds latency. This is the classic 'blocking I/O on a ROS callback thread' anti-pattern.

**Evidence.** Subscription creation at 405-408 passes `sub_options` only (no callback group) vs. the timers which explicitly pass callback_group_* (357,363,379,384). write_buffer issues a synchronous libusb_bulk_transfer with timeout_ms_ (usb.cpp 565-571); on partial/failed transfer it throws UsbException (usb.cpp 581-593) caught at node 1571. Note other device-input callbacks (esf_meas/rxm_*) use send_async() (e.g. 1447,1511,1524) i.e. non-blocking submit, so rtcm is the lone synchronous-blocking writer.

**Repro.** Stall the OUT endpoint (device busy / link renegotiation) while RTCM streams; observe other default-group callbacks (parameter set, other input subs) stop progressing for up to 250ms per message and RTCM backlog growth.

**Suggested fix.** Either (a) route RTCM out through the existing async path (write_buffer_async / a queued out-path) instead of the blocking write_buffer, or (b) give rtcm_sub_ its own dedicated MutuallyExclusive callback group so a stalled write cannot head-of-line-block unrelated callbacks, and reduce the OUT timeout. Prefer (a).

**Verification.** Verified all cited code. rtcm_sub_ (ublox_dgnss_node.cpp:405-408) is created with `sub_options` only, no callback group, so it lives in the node default MutuallyExclusive group. rtcm_callback (1541-1574) calls blocking usbc_->write_buffer at 1570. write_buffer (usb.cpp:565-571) does a synchronous libusb_bulk_transfer with timeout_ms_=250 (usb.cpp:44) and throws UsbException on rc<0 or short transfer (581-593), caught at node 1571-1573. Sibling input callbacks all use non-blocking send_async() (1447 esf_meas, 1511 pmp, 1524 qzssl6, 1537 spartnkey), so rtcm is indeed the lone synchronous blocking writer. The four timers (ubx/rtcm/usb_events/param_processing) each get a dedicated MutuallyExclusive group (node.cpp:131-138, 357/363/379/384). So the blocking-I/O-on-callback-thread anti-pattern and intra-group head-of-line blocking among the input subscriptions are REAL.

Severity correction P1 -> P2. Decisive fact the finding omitted: ALL 16 launch files use executable='component_container_mt' (a MultiThreadedExecutor; grep confirms 16x component_container_mt, 0 single-threaded). Under MT …

---

### NODE-06 — Reconnection path (hotplug_attach) sets device_attached_/READY and restores params without re-validating, and is_reconnection vs reset_device_parameters can race a still-in-flight detach

- **Severity:** P2  **Probability:** medium  **Class:** data-race / correctness  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:1727-1763`

**What's wrong.** hotplug_attach_callback (1727) sets device_attached_=true (1729) and unconditionally device_readiness_state_=READY (1742/1746) after perform_usb_initialization(), even though perform_usb_initialization can return early without a usable device (devh_valid() false -> returns at node 477) or can have called rclcpp::shutdown() on failure (502) — yet the caller still marks READY and logs 're-connection completed' (1742-1743). device_attached_ (node 619) and has_been_connected_before_ (627) are plain bools written from the events thread and read elsewhere with no synchronization. hotplug_detach_callback (1753) concurrently sets device_attached_=false (1756) and calls parameter_manager_->reset_device_parameters() (1761) which invalidates non-USER params under param_cache_mutex_ (parameters.cpp 235). A rapid detach->attach can interleave reset_device_parameters (clearing params to PARAM_INITIAL) with the attach-side parameter restoration driven off handle_param_processing_callback, producing lost or double-sent parameter writes. The READY flag is set before any confirmation that the device actually re-enumerated correctly.

**Evidence.** hotplug_attach: device_attached_=true (1729), is_reconnection=has_been_connected_before_ (1731), perform_usb_initialization() (1740) can early-return at node 477 or shutdown at 502 yet code still sets device_readiness_state_=READY (1742). hotplug_detach: device_attached_=false (1756), reset_device_parameters() (1761) -> parameters.cpp 233-262 mutates param_cache_map_ resetting to PARAM_INITIAL/needs refetch. device_attached_/has_been_connected_before_ are plain bools (node 619,627). Both attach and detach run on the libusb events thread but param restore runs on param_processing_timer_ (separate group, node 384).

**Repro.** Bounce the cable quickly (detach then attach within the init window). Observe READY set despite a failed/partial init, and parameter restore racing reset_device_parameters leaving device with stale or unapplied config.

**Suggested fix.** Only set device_readiness_state_=READY after perform_usb_initialization confirms devh_valid()/init_async succeeded (propagate a success bool from perform_usb_initialization). Guard device_attached_/has_been_connected_before_ with the same lock as the param cache or make atomic, and sequence reset_device_parameters vs parameter restore (e.g. version/generation counter) so a stale detach cannot wipe a fresh attach's restored params.

**Verification.** Read all cited code. The finding is PARTIALLY real: one genuine defect exists at the cited lines, but the headline data-race/param-loss mechanism is refuted.

REAL defect (ublox_dgnss_node.cpp:1740-1743): hotplug_attach_callback calls perform_usb_initialization() (1740) then UNCONDITIONALLY sets device_readiness_state_=READY (1742) and logs "Hotplug device re-connection completed" (1743). Verified perform_usb_initialization (462-525) can early-return at 477 when !devh_valid() (no usable device) OR catch an exception and call rclcpp::shutdown() at 502/509/516/523 — in all those cases the attach callback still marks READY and logs success with no success propagation. This is a correctness/diagnosability robustness issue. P2 is appropriate.

REFUTED claim 1 (unsynchronized cross-thread reads of device_attached_/has_been_connected_before_): FALSE. grep of the whole node shows device_attached_ (decl 619) is written at 1729/1756/145 and NEVER read anywhere; has_been_connected_before_ (627) is read only at 1731, on the same events thread that writes it (1747). Neither has a cross-thread rea …

---

### PARAM-01 — Atomic CFG-VALSET batch: one unsupported key NAKs the whole batch, silently disabling ALL config; NAK not logged with offending key, no per-key fallback/bisect

- **Severity:** P2  **Probability:** high  **Class:** config-fragility  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:1788-1884 (batch assembly/send); 2492-2513 (NAK handling)`

**What's wrong.** send_parameters_to_device_batch() assembles all user/restore parameters into a single CFG-VALSET payload and sends it transactionless (cfg_val_set_transaction(0); cfg_val_set_poll_async()). The ublox device validates a VALSET as a unit: if ANY single key is unsupported on the connected device family (e.g. CFG_UART1OUTPROT_RTCM3X / CFG_UART2* keys present in a generic config but unsupported on ZED-F9R), the device returns ONE UBX-ACK-NAK for the entire CFG-VALSET and applies NONE of the keys. Result: every message-rate / output / protocol setting in that batch is silently dropped, so the node looks configured but emits nothing. The NAK handler at 2492-2513 only logs the generic ack-nak payload (msg_class/msg_id of CFG-VALSET = 0x06/0x8a) at WARN; the per-key retry/bisect logic is entirely commented out (2501-2512). There is NO indication of which key failed and NO per-key or bisect fallback. The flush-every-10 loop (1845-1851) sends sub-batches but each sub-batch is still all-or-nothing, so a bad key still kills its 10-key chunk. Note the AckNakPayload (ubx_ack.hpp:74-81) carries only msg_class+msg_id, so the offending key is genuinely not recoverable from the NAK alone — bisect/per-key send is the only way to identify it.

**Evidence.** 1840 ubx_cfg_->cfg_val_set_key_append(cfg_item->ubx_key_id, value); accumulates all keys into one payload. 1848-1849 cfg_val_set_transaction(0); cfg_val_set_poll_async(); sends them as one frame. 1862-1866 sends the remainder as one frame. 2496-2499 RCLCPP_WARN logs only payload_ack_nak->to_string() (class/id), not any key name. 2500-2512 the retry-on-NAK block is commented out. parameters.cpp send_batch_parameters (136-164) treats the device_batch_callback bool as success/fail for the WHOLE vector and on failure simply leaves needs_device_send untouched with a generic 'Failed to send batch of N parameters' log — no per-key attribution.

**Repro.** On a ZED-F9R, supply a params/yaml (or default TOML) that includes a key unsupported by F9R (e.g. CFG_UART1OUTPROT_RTCM3X or a CFG_UART2* key). At startup the whole CFG-VALSET batch is NAK'd; logs show only 'ack nak payload - class:0x06 id:0x8a'; none of the message-output config is applied; topics stay silent. No log names the offending key.

**Suggested fix.** On UBX-ACK-NAK for CFG-VALSET, implement a bisect-on-NAK: re-send the failing batch split in halves until isolating the offending key(s), then log each offending key BY NAME (resolve via ParameterManager::find_config_item_by_key) at WARN and re-send the remaining good keys so valid config is still applied. Alternatively send each key as its own CFG-VALSET when a batch NAKs. Track the in-flight key set per VALSET frame so the NAK handler can correlate (the commented-out cfg_val_set_frame() machinery is the hook). At minimum, on NAK enumerate and log the names of the keys that were in the just-sent batch.

**Verification.** Verified all cited code in /home/gtec/ublox_dgnss_audit. Batch send (ublox_dgnss_node.cpp:1788-1884): keys accumulated via cfg_val_set_key_append (1840), flushed transactionless every 10 keys (1845-1851) and remainder (1862-1866) via cfg_val_set_transaction(0)+cfg_val_set_poll_async(). Each VALSET frame is all-or-nothing on the device, so any unsupported key NAKs its frame and applies none of that frame's keys. NAK handler (2492-2513) logs only payload_ack_nak->to_string(); confirmed in ubx_ack.hpp: to_string (57-59) emits only class/id, and AckNakPayload (74-83) parses only msg_class+msg_id from a 2-byte payload, so the offending key is genuinely unrecoverable from the NAK. The CFG-VALSET retry/bisect block (2500-2512) is fully commented out. parameters.cpp send_batch_parameters (136-164) treats the callback bool as whole-vector success/fail; on failure it leaves needs_device_send untouched with a generic 'Failed to send batch of N' log, no per-key attribution. CORRECTIONS: (1) The description overstates impact: due to the flush-every-10 chunking, a single bad key kills only its 10- …

---

### PARAM-02 — Data race on shared UbxCfg cfg_valget_ frame: VALGET response parsed on ubx_timer thread without cfg_batch_mutex_, racing the request build under the lock

- **Severity:** P2  **Probability:** medium  **Class:** data-race  **Verify confidence:** medium
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:2524-2540 (ubx_cfg_in_frame, no lock); 3962-4032 (ublox_fetch_device_params_async, under cfg_batch_mutex_)`

**What's wrong.** UbxCfg holds shared mutable state (cfg_valget_ for VALGET, valset_payload_poll_/valset_frame_poll_ for VALSET) with NO internal locking (ubx_cfg.hpp:614-651). The node guards VALSET/VALGET *request* assembly with cfg_batch_mutex_ (1789, 3972, 3905, 4145, 4213, 4262) which run on the param-processing / usb-init / service threads. But the VALGET *response* path runs on the ubx_timer callback group: ubx_timer_callback -> ubx_cfg_in_frame -> set_cfg_val_get_frame() (writes cfg_valget_->frame) and cfg_val_get_payload()->to_string()/parse (reads/mutates cfg_valget_), with NO cfg_batch_mutex_ held. Because each timer uses a distinct MutuallyExclusive callback group (132-138, 357/384), a MultiThreadedExecutor (the normal case for a composable component / container) runs these concurrently. So while ublox_fetch_device_params_async (holding cfg_batch_mutex_) is calling cfg_val_get_key_append / make_frame_poll on cfg_valget_, the ubx_timer thread can be calling set_cfg_val_get_frame()/cfg_val_get_payload() on the same object: concurrent read+write of the same shared frame/payload buffers => undefined behaviour (torn reads, use of a being-rebuilt buffer).

**Evidence.** 2528 ubx_cfg_->set_cfg_val_get_frame(f->ubx_frame); and 2532 ubx_cfg_->cfg_val_get_payload()->to_string() execute inside ubx_cfg_in_frame, reached from ubx_timer_callback (1978) via 2120 ubx_cfg_in_frame(f) with NO cfg_batch_mutex_ anywhere on this path. 3972 std::lock_guard lock(cfg_batch_mutex_) protects the request build which calls 3999 cfg_val_get_key_append and 4015/4025 cfg_val_get_poll_async_all_layers() on the SAME cfg_valget_. ubx_cfg.hpp:614-623/640-651 show no internal mutex. Callback groups are independent MutuallyExclusive (132-138).

**Repro.** Run the node in a MultiThreadedExecutor / component container, trigger a device-param fetch (ublox_fetch_device_params_async) while CFG-VALGET responses are streaming back and being parsed on the ubx_timer thread. Under ThreadSanitizer this reports a data race on cfg_valget_ / the valget payload buffer; in the field it manifests as occasional corrupted/dropped param reads or crashes during fetch.

**Suggested fix.** Protect ALL access to the shared UbxCfg state with cfg_batch_mutex_, including the VALGET response path: take std::lock_guard(cfg_batch_mutex_) in ubx_cfg_in_frame around set_cfg_val_get_frame()/cfg_val_get_payload()/ubx_cfg_payload_parameters(). Better: give UbxCfg its own internal mutex so request build and response parse are serialized regardless of caller, since the same object's mutable frame buffers are shared across threads.

**Verification.** Verified the data race exists, but mechanism is partly overstated; downgrading P1 -> P2.

CONFIRMED facts:
- Response path has NO lock: ubx_timer_callback (1978, callback_group_ubx_timer_ at 357) -> ubx_queue_frame_in (2103) -> ubx_cfg_in_frame (2524-2541): set_cfg_val_get_frame (2528 writes cfg_valget_->frame_) and cfg_val_get_payload()->to_string()/parse (2532-2533). No cfg_batch_mutex_ anywhere on this path.
- Request path IS under lock: ublox_fetch_device_params_async (3962) holds std::lock_guard(cfg_batch_mutex_) at 3972, calls cfg_val_get_key_append (3999), cfg_val_get_poll_async_all_layers (4015/4025 -> make_frame_poll). Reached from perform_usb_initialization (462/487) -> ublox_dgnss_init_async (4316) -> ublox_init_all_cfg_items_async (3878/4344) -> 3894. Runs on usb_init_timer_ (350) which has NO explicit callback group -> default MutuallyExclusive group, distinct from callback_group_ubx_timer_.
- Distinct MutuallyExclusive callback groups confirmed (131-138, 357). Registered composable component (RCLCPP_COMPONENTS_REGISTER_NODE 4381), so a MultiThreadedExecutor container is …

---

### PARAM-03 — CFG-VALGET completion uses 'no params left in PARAM_VALGET' as success; unsupported/omitted keys and failed async writes stay PARAM_VALGET forever (only resolved by blanket timeout), with no ACK/NAK correlation per key

- **Severity:** P2  **Probability:** high  **Class:** decode-gap  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:3998-4002 (mark VALGET before write); 4041-4055 (completion test); 966-974 (response demux)`

**What's wrong.** The fetch state machine marks each key PARAM_VALGET (4002) BEFORE the async poll is actually written (4015/4025 cfg_val_get_poll_async_all_layers -> write_buffer_async with NULL callback). A key only transitions out of PARAM_VALGET when a matching VALGET response arrives and ubx_cfg_payload_parameters() (966-974) finds the key via find_config_item_by_key and calls update_param_from_device -> PARAM_LOADED. There is no ACK/NAK correlation for VALGET: the device omits unsupported/unset keys from the response entirely (or NAKs the whole poll, handled only generically at 2492-2513). Consequently any key the device does not return — unsupported on this family, not set in the queried layer, or whose async write silently failed — remains PARAM_VALGET indefinitely. check_param_fetch_completion (4041-4055) treats valget_count==0 as the ONLY success condition, so a single never-answered key blocks completion and device readiness until PARAM_FETCH_TIMEOUT fires (4058+), then logs them as 'stuck'/'Missing response' but cannot say whether the key was unsupported vs lost. Marking PARAM_VALGET before the write also means a write failure leaves the key permanently stuck. Matching is purely key-id based with no sequence/echo check, so a late/duplicate VALGET response for a key re-requested in a later cycle can be mis-attributed.

**Evidence.** 4002 update_parameter_status(ubx_ci.ubx_config_item, PARAM_VALGET) executes inside the iterate lambda BEFORE 4015/4025 cfg_val_get_poll_async_all_layers(). 4041 valget_count = count_parameters_by_status(PARAM_VALGET); 4043 if (valget_count == 0) is the sole success path. 968-973 only keys present in cfg_val_get_payload->cfg_data are cleared to PARAM_LOADED; omitted keys are never touched. 4058-4069 timeout path logs 'stuck in PARAM_VALGET' / 'Missing response' with no NAK/unsupported distinction.

**Repro.** Use a config TOML/include list containing a key the device firmware does not implement. The poll never returns that key; it stays PARAM_VALGET; device_readiness_state_ never becomes READY until PARAM_FETCH_TIMEOUT, after which it is reported as a missing response with no indication it was actually unsupported.

**Suggested fix.** Mark keys PARAM_VALGET only AFTER a successful async write (or use the write completion callback instead of passing NULL). On completion, distinguish unsupported/omitted keys from lost ones: after a bounded number of poll cycles, demote still-PARAM_VALGET keys to a PARAM_UNSUPPORTED/absent state and proceed to READY instead of blocking on valget_count==0. Correlate VALGET responses against the set of keys actually requested in the current cycle (and handle CFG-VALGET NAK explicitly) so late/duplicate responses are not mis-attributed.

**Verification.** Verified against live source at /home/gtec/ublox_dgnss_audit. All cited lines accurate. (1) ublox_dgnss_node.cpp:4002 sets PARAM_VALGET inside the iterate lambda BEFORE the async poll at 4015/4023/4025 — so a write that fails leaves the key permanently PARAM_VALGET. (2) The poll is genuinely fire-and-forget: cfg_val_get_poll_async_all_layers -> cfg_val_get_poll_async -> write_buffer_async(..., NULL) at ubx_cfg.hpp:622; no write-completion callback. (3) ubx_cfg_payload_parameters (966-974) only clears keys actually present in cfg_data via find_config_item_by_key->update_param_from_device->PARAM_LOADED; omitted/unsupported keys are never touched, and matching is key-id only with no sequence/echo check, so a late/duplicate VALGET for a re-requested key can be mis-attributed. (4) check_param_fetch_completion (4043) uses valget_count==0 as the ONLY success path; a single unanswered key blocks DeviceReadinessState::READY until PARAM_FETCH_TIMEOUT. (5) Confirmed no PARAM_UNSUPPORTED state exists — parameters.hpp:60-66 enum only has PARAM_INITIAL/USER/LOADED/VALSET/VALGET — so timeout path ( …

---

### UBX-03 — RAWX decoder trusts num_meas and reads 32*num_meas bytes with no payload-length bound

- **Severity:** P2  **Probability:** medium  **Class:** buffer-over-read  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/rxm/ubx_rxm_rawx.hpp:104-130`

**What's wrong.** RxmRawxPayload reads num_meas from payload byte 11, then loops i in [0,num_meas) reading 32 bytes per iteration via buf_offset<T>() starting at offset 16. buf_offset (utils.hpp:193-199) does a raw `memcpy(&value, buf->data()+offset, sizeof(T))` with NO bounds check. The payload_ vector is sized to `size` (the frame length), but there is no check that size >= 16 + 32*num_meas. A frame whose length field / num_meas disagree (num_meas large, payload short) reads far past the heap buffer. num_meas is a single wire byte (0-255 → up to 8176 bytes read past a possibly-tiny payload). This is reachable for any RXM-RAWX frame since SFRBX/RAWX ARE dispatched (node.cpp:2784-2786).

**Evidence.** rawx.hpp:104 `num_meas = buf_offset<u1_t>(&payload_, 11);` then loop 111-130 with `offset += 32` and unguarded buf_offset calls; utils.hpp:197 `memcpy(&value, buf->data() + offset, sizeof(T));` no bounds check.

**Repro.** Inject a valid-checksum RXM-RAWX (0x02 0x15) frame with payload length 16 but num_meas=255; decoder reads ~8KB past the 16-byte buffer.

**Suggested fix.** Before the loop verify `payload_.size() >= 16 + 32u*num_meas` (and >= 16 for the header) and clamp/reject otherwise. Make buf_offset bounds-check and throw on overflow.

**Verification.** VERIFIED the code mechanics exactly as described, but downgrading severity P1 -> P2 because reachability is gated by a checksum the finding omitted.

Code confirmed:
- ublox_dgnss_node/include/ublox_dgnss_node/ubx/rxm/ubx_rxm_rawx.hpp:104 reads num_meas = buf_offset<u1_t>(&payload_,11); loop 111-130 reads offset 16 + 32*i via buf_offset, offset+=32. With num_meas up to 255 the loop touches offset 16+32*254+30 = 8174 (~8176 bytes).
- utils.hpp:193-199 buf_offset does raw `memcpy(&value, buf->data()+offset, sizeof(T))` with NO bounds check (confirmed line 197). There is no validation that payload_.size() >= 16 + 32*num_meas anywhere.
- Two distinct over-reads exist: (a) the ctor's own memcpy at rawx.hpp:99 copies `size`(=frame->length) bytes from frame->payload, and (b) the num_meas loop reads past payload_ when length < 16+32*num_meas.

Plumbing confirmed: ubx.hpp:69 `length = *reinterpret_cast<u2_t*>(&buf[4])` is the wire-claimed length (NOT clamped to received bytes); ubx.hpp:70 payload=&buf[6]; FrameContainer::frame (ubx.hpp:382) passes frame->length as size into Payload<RxmRawxPay …

---

### UBX-04 — SFRBX decoder trusts num_words and reads 4*num_words past payload

- **Severity:** P2  **Probability:** medium  **Class:** buffer-over-read  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/rxm/ubx_rxm_sfrbx.hpp:62-71`

**What's wrong.** RxmSfrbxPayload reads num_words from payload byte 4, then loops i in [0,num_words) reading u4_t at offset 8,12,16,... via the unchecked buf_offset memcpy. No verification that payload size == 8 + 4*num_words. A crafted/corrupted SFRBX with num_words=255 reads 1020 bytes; if the actual payload is short, this over-reads the heap. Note: this directly refutes the audit's hypothesis that 0x02 0x13 (SFRBX) falls through to a no-op branch — it is fully dispatched at node.cpp:2788-2790 and decoded here, so the OOB read is live, not dead code.

**Evidence.** sfrbx.hpp:62 `num_words = buf_offset<u1_t>(&payload_, 4);` then 67-71 `size_t offset = 8; for(i<num_words) dwrd.push_back(buf_offset<u4_t>(&payload_, offset)); offset+=4;` with unchecked buf_offset (utils.hpp:197). Dispatch confirmed live at node.cpp:2788.

**Repro.** Inject checksum-valid RXM-SFRBX (0x02 0x13), payload length 8, num_words=255 → reads ~1KB past buffer.

**Suggested fix.** Validate `payload_.size() >= 8 + 4u*num_words` before the loop; reject or clamp. Bounds-check buf_offset.

**Verification.** Verified real, line-level. ubx_rxm_sfrbx.hpp:62 reads num_words = buf_offset<u1_t>(&payload_,4); lines 66-71 loop i in [0,num_words) reading buf_offset<u4_t>(&payload_, offset) at offset 8,12,... with NO check that payload_.size() >= 8 + 4*num_words. buf_offset (utils.hpp:194-199) is an unchecked memcpy(&value, buf->data()+offset, sizeof(T)) — confirmed no bounds check. Dispatch is live, not dead: node.cpp:2788-2790 fully dispatches UBX_RXM_SFRBX to sfrbx()->frame()+pub. 

Confirmed the over-read is NOT guarded upstream: ublox_in_callback (node.cpp:1631-1636) builds the frame from len = actual USB bytes received, then from_buf_build() (ubx.hpp:63-73) sets length = *(u2_t*)&buf[4] (the DECLARED wire length) and payload = &buf[6]. There is NO checksum verification and NO check that len == 8+length before the frame is queued/decoded. FrameContainer::frame (ubx.hpp:382) then does Payload<T>(frame->payload, frame->length); the SFRBX ctor (sfrbx.hpp:53-56) resizes payload_ to size=length and memcpys length bytes, so the loop over-reads the payload_ heap vector whenever byte-4 num_words is …

---

### UBX-05 — Single USB transfer assumed to be exactly one UBX frame — extra/partial frames silently dropped

- **Severity:** P2  **Probability:** high  **Class:** decode-gap  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/ublox_dgnss_node.cpp:1619-1656`

**What's wrong.** ublox_in_callback treats one libusb bulk-in transfer as exactly one message: it checks buf[0]/buf[1] for sync chars (or 0x24 NMEA, or 0xD3 RTCM) and copies the ENTIRE `len` bytes into a single Frame, with no UBX framing/reassembly loop. If the device packs multiple UBX messages into one bulk transfer (common on high-rate output: NAV-PVT + NAV-SAT + RXM-RAWX etc. in one 64-byte-multiple transfer), only the first message's class/id is honored and the whole blob is copied into one Frame whose `length` field is the first message's length — the trailing messages are silently lost (and the checksum check on the oversized buf will usually fail and drop the whole thing). Conversely a single UBX message larger than one bulk transfer (split across transfers) is never reassembled; each partial transfer either fails sync detection or is treated as a standalone frame. There is no ring-buffer / streaming UBX parser. Result: silent data loss and corrupted decode under realistic multi-message transfers.

**Evidence.** node.cpp:1631-1641 copies whole `len` into one frame->buf then from_buf_build (single message). No loop over multiple UBX records, no carry-over buffer for partial frames between callbacks. The frame length field (buf[4..5]) is never compared to len to detect 'more bytes follow' or 'frame incomplete'.

**Repro.** Enable several high-rate output messages so the device coalesces them into one bulk-in; observe only the first decodes and/or checksum-failure drops in logs.

**Suggested fix.** Implement a persistent byte-stream accumulator: append each transfer to a member buffer, then repeatedly scan for sync chars, read the length field, wait until 6+length+2 bytes are present, validate checksum, emit one Frame, and advance. Handle multiple frames per transfer and frames spanning transfers.

**Verification.** Confirmed by reading the cited code. ublox_in_callback (ublox_dgnss_node.cpp:1619-1656) treats one bulk-in transfer as exactly one UBX message: it inspects only buf[0]/buf[1], memcpy's the ENTIRE actual_length into a single Frame, and calls from_buf_build() once (line 1631-1641). There is no framing/reassembly loop and no carry-over buffer between callbacks. The transfer-in buffer is large (IN_BUFFER_SIZE = 64*200 = 12800, usb.hpp:48) and make_transfer_in (usb.cpp:753-771) sets NO flags — notably not LIBUSB_TRANSFER_SHORT_NOT_OK (that flag is only on the OUT transfer, usb.cpp:741) — so back-to-back UBX messages emitted close together can be coalesced by the host controller into a single transfer.

Verified the downstream failure mode precisely: from_buf_build (ubx.hpp:63-73) sets length = buf[4..5] (first frame's payload len) but sets ck_a/ck_b = buf[size-2]/buf[size-1] (last two bytes of the WHOLE concatenated blob). ubx_check_sum (ubx.hpp:75-87) calls build_frame_buf() which rebuilds only 8+length bytes (first frame) and computes the checksum over that; ubx_frame_checksum_check (no …

---

### UBX-07 — Polled-frame checksum check uses AND instead of OR — accepts frames with one wrong checksum byte

- **Severity:** P2  **Probability:** medium  **Class:** checksum-validation  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx.hpp:150`

**What's wrong.** In get_polled_frame the checksum test is `if (ck_a != polled_frame->ck_a && ck_b != polled_frame->ck_b) throw`. With && the exception fires only when BOTH bytes mismatch; a frame where exactly one of ck_a/ck_b is correct passes validation. The correct UBX check is OR (reject if either byte mismatches). This weakens the synchronous poll-path integrity check (note the async path ubx_frame_checksum_check at node.cpp:2096 correctly uses ||, so the two paths disagree). A single-byte corruption on the polled config/ACK path is accepted as valid.

**Evidence.** ubx.hpp:150 `if (ck_a != polled_frame->ck_a && ck_b != polled_frame->ck_b)`; contrast node.cpp:2096 `if (ck_a != ubx_f->ck_a || ck_b != ubx_f->ck_b)` (correct).

**Repro.** Polled response with correct ck_a but wrong ck_b is accepted instead of throwing UbxAckNackException.

**Suggested fix.** Change && to || at ubx.hpp:150 to reject when either checksum byte differs.

**Verification.** Confirmed by reading the actual source. ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx.hpp:150 reads exactly `if (ck_a != polled_frame->ck_a && ck_b != polled_frame->ck_b) { throw UbxAckNackException("polled frame checksum failed"); }`. With &&, the exception only fires when BOTH checksum bytes are wrong; a polled frame in which exactly one of ck_a/ck_b is correct passes validation. This is in get_polled_frame, the synchronous config/ACK poll path. The contrast is verified: node.cpp:2096 ubx_frame_checksum_check uses the correct `if (ck_a != ubx_f->ck_a || ck_b != ubx_f->ck_b)` (OR), so the two paths genuinely disagree. The correct UBX check rejects when either byte differs, so the && is a real logic bug. file:line, mechanism, evidence, and suggested fix (change && to ||) are all accurate. Severity P2 is honest: this weakens integrity/diagnosability on the poll path but does not cause memory corruption, crash, or the production wedge; it is also partly masked by USB bulk-transfer CRC, and the trigger (single-byte UBX corruption leaving exactly one checksum byte matching) is narrow …

---

### USB-02 — no_device_streak_ / attached_ escalation covers only NO_DEVICE, never TIMEOUT or persistent transfer errors

- **Severity:** P2  **Probability:** high  **Class:** wedge-recovery  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/usb.cpp:952-964`

**What's wrong.** The only mechanism that flips attached_ to false outside an explicit DEVICE_LEFT hotplug is the no_device_streak_ counter, and it is incremented exclusively inside `case LIBUSB_ERROR_NO_DEVICE` (952-956). Every successful or benign libusb_handle_events_timeout return (rc>=0) RESETS the streak to 0 (962-964). Because handle_events keeps returning >=0 while a transfer silently times out/stalls, the streak never builds. So the kNoDeviceThreshold=3 (usb.hpp:156) escalation that would clear attached_ — which downstream gates (rtcm_callback node.cpp:1550, handle_param_processing node.cpp:1314) and would allow re-arming — is unreachable for the TIMEOUT/STALL class of failure. The device family transition CONNECTED->ERROR/DISCONNECTED in driver_state_ is likewise never driven by transfer-level timeouts.

**Evidence.** usb.cpp:953 `if (++no_device_streak_ >= kNoDeviceThreshold) { attached_ = false; }` is inside `case LIBUSB_ERROR_NO_DEVICE:`. usb.cpp:962 `if (rc >= 0) { no_device_streak_ = 0; }` resets on every healthy poll. No code path increments no_device_streak_ on TIMED_OUT/STALL/ERROR transfer statuses (callback_in 660-692 never touches it).

**Suggested fix.** Track transfer-completion liveness independently of handle_events rc. Increment a separate stall counter on consecutive non-COMPLETED IN statuses in callback_in and escalate attached_/driver_state_ when it crosses a threshold, mirroring the NO_DEVICE logic, so the existing rtcm/param gates and re-arm logic actually engage during a silent wedge.

**Verification.** All factual claims verified against live source at /home/gtec/ublox_dgnss_audit/ublox_dgnss_node/src/usb.cpp. Confirmed: (1) no_device_streak_ increments ONLY inside case LIBUSB_ERROR_NO_DEVICE (usb.cpp:953); (2) rc>=0 resets streak to 0 every benign poll (usb.cpp:962-964); (3) callback_in (usb.cpp:655-715) handles TIMED_OUT/STALL/ERROR only via exception_cb_fn_ capped at err_count_<10 (689) and never touches no_device_streak_, attached_, or driver_state_; (4) kNoDeviceThreshold=3 at usb.hpp:156; (5) driver_state_ -> ERROR only at usb.cpp:259 during connect, never from transfer timeouts. Downstream gates confirmed: rtcm_callback reads attached() at node.cpp:1550, handle_param_processing reads driver_state() at node.cpp:1314. So the escalation-unreachable-for-TIMEOUT/STALL mechanism is REAL and correctly described; bug exists.

Severity correction P0 -> P2. The finding overstates impact: it asserts the escalation 'would allow re-arming.' Code refutes this. Setting attached_=false does NOT trigger any re-open. The only re-open path is hotplug_attach_callback (usb.cpp:477-486), driven e …

---

### USB-06 — transfer_queue / cleanup functions accessed without consistent locking; queued_transfer_in_num and submit_transfer race against callbacks

- **Severity:** P2  **Probability:** medium  **Class:** data-race  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/usb.cpp:793-908, 644, 706, 974-975`

**What's wrong.** transfer_queue_ is guarded by transfer_queue_mutex_ inconsistently. cleanup_transfer_queue checks transfer_queue_.size()==0 (805) and reads debug_cb_fn_ BEFORE taking the lock (811), and submit_transfer calls cleanup_transfer_queue (799) right after releasing the push_back lock (794-796) — so size() at 805 is read without the lock. queued_transfer_in_num likewise reads .size() at 895 before locking at 897 and iterates the deque. close_devh calls cleanup_transfer_queue then cleanup_all_transfers (974-975). While all of this nominally runs on the single event thread for the resubmit/cancel paths, submit_transfer is ALSO called from write_buffer_async (usb.cpp:728) on the subscription/ubx/param threads (node.cpp:1447,1511,1570 via async), so push_back at 795 and the lock-free size()/iteration in cleanup_transfer_queue(805)/queued_transfer_in_num(895) genuinely race: a concurrent push_back can reallocate the deque while the event thread iterates it in cleanup, corrupting iterators (use-after-free / crash). The completed flag is a plain bool written in callbacks (639,701) and read under different conditions.

**Evidence.** usb.cpp:805 `if (transfer_queue_.size() == 0) {return;}` then lock at 811. usb.cpp:895 `if (transfer_queue_.size() == 0) {return 0;}` then lock at 897. submit_transfer push_back under lock 794-796 but then cleanup_transfer_queue() at 799 re-locks; write_buffer_async->submit_transfer (728) runs on non-event threads. `bool completed` is non-atomic (usb.hpp:68), written at usb.cpp:639/701 in callbacks and read at 816/903 elsewhere.

**Suggested fix.** Take transfer_queue_mutex_ before the size() short-circuit in both cleanup_transfer_queue and queued_transfer_in_num (or make those checks part of the locked section). Ensure every push_back/erase/iteration of transfer_queue_ is under the mutex with no lock-free fast paths. Make transfer_t::completed std::atomic<bool> or only read it under the queue lock.

**Verification.** Confirmed real data race, with mechanism/citation corrections.

THREADING (verified): Launch uses executable="component_container_mt" (ublox_dgnss/launch/ublox_rover_hpposllh.launch.py:33, ublox_fb+r_rover.launch.py:62) => MultiThreaded executor. The usb-events 10ms timer is in its own MutuallyExclusive group callback_group_usb_events_timer_ (node.cpp:134,379), distinct from the node-default group used by the input subscriptions (sub_options at node.cpp:386-387 sets NO callback_group) and from callback_group_ubx_timer_/param groups. So handle_usb_events (node.cpp:1283) -> callback_in/out -> submit_transfer/cleanup_transfer_queue/queued_transfer_in_num genuinely runs concurrently with write_buffer_async->submit_transfer driven by subscription/ubx/param callbacks. Concurrency is real.

REAL RACE (verified): cleanup_transfer_queue reads transfer_queue_.size() at usb.cpp:805 BEFORE taking transfer_queue_mutex_ at 811; queued_transfer_in_num reads .size() at 895 before locking at 897. These unlocked std::deque::size() reads race with push_back (795) and erase (852) on other threads -> UB. …

---

### USB-08 — hotplug_attach_callback can open a second device / re-enter open_device with no locking; driver_state_ races and attach during active streaming

- **Severity:** P2  **Probability:** medium  **Class:** data-race  **Verify confidence:** medium
- **Location:** `ublox_dgnss_node/src/usb.cpp:468-505, 242-262`

**What's wrong.** hotplug_attach_callback guards on `if (!attached_)` (477) then calls open_device() (478) which overwrites devh_, dev_, connected_product_id_, num_interfaces_, ep_* (usb.cpp:247,275,284,298,363-368). attached_ is a plain bool read at 477 and written at 481 with no memory barrier. Both attach and detach callbacks run from the event thread, so they are mutually serialized there — but open_device() writes devh_/ep_* that the subscription/ubx threads read concurrently in make_transfer_out (744) and write_buffer (565) with no lock (see USB-03). The ENUMERATE flag (usb.cpp:96,114) means the attach callback also fires synchronously during init() registration, potentially before in/out callbacks and queue state are fully wired, and driver_state_ is a non-atomic enum (usb.hpp:147) written here (479) and read on other threads in node.cpp (1269,1276,1314,1338). On a quick detach/reattach, attached_ may still read true (event thread hasn't processed DEVICE_LEFT yet) causing a re-attach to be skipped, or ep addresses to change under a concurrent in-flight write.

**Evidence.** usb.cpp:477 `if (!attached_)` then 481 `attached_ = true;` non-atomic. open_device reassigns devh_ (247), dev_ (275), ep_data_in_addr_/ep_data_out_addr_ (363-381) read elsewhere without lock. driver_state_ enum (usb.hpp:147) written at 479/499 read across threads at node.cpp:1269/1276/1314/1338. Hotplug registered with LIBUSB_HOTPLUG_ENUMERATE (usb.cpp:96,114) firing attach synchronously at registration time.

**Suggested fix.** Guard all of devh_/dev_/ep_*/attached_/driver_state_ updates in open_device/hotplug callbacks under the same state mutex proposed in USB-03, and make driver_state_ access atomic or lock-protected. Quiesce/cancel any in-flight transfers and block async writers before mutating ep addresses on re-attach.

**Verification.** Verified against live source at /home/gtec/ublox_dgnss_audit.

CONFIRMED TRUE:
- attached_ (usb.hpp:145) and driver_state_ (usb.hpp:147) are plain non-atomic members.
- hotplug_attach_callback (usb.cpp:468-487): reads !attached_ at 477, calls open_device() at 478, sets driver_state_=CONNECTED at 479, attached_=true at 481 — no lock/barrier.
- open_device() (usb.cpp:242-388) reassigns devh_(247), dev_(275), connected_product_id_(284), num_interfaces_(298), ep_data_out_addr_/ep_data_in_addr_/ep_comms_in_addr_(363-368,377-381) with no lock.
- write_buffer reads devh_/ep_data_out_addr_ under write_mutex_ (563-567), but the attach callback mutating those fields does NOT hold write_mutex_, so the lock does not serialize against the mutator. Real synchronization gap.
- rtcm_callback (node.cpp:1541) checks dev_valid()/attached() without a lock (1543,1550) then calls blocking write_buffer (1570).
- LIBUSB_HOTPLUG_ENUMERATE registered (usb.cpp:96,114) -> attach callback fires synchronously at init() registration.
- driver_state() read on threads at node.cpp:1269,1276,1314,1338 confirmed.
- Con …

---

## P3 findings (3)

### UBX-06 — buf_offset offset parameter is uint16_t — truncates for payloads >64KB

- **Severity:** P3  **Probability:** low  **Class:** integer-truncation  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/include/ublox_dgnss_node/ubx/utils.hpp:193-199`

**What's wrong.** buf_offset<T>(std::vector<u1_t>*, u2_t offset) takes the offset as u2_t (uint16_t). Callers in rawx.hpp pass `size_t offset` (line 110, incrementing by 32) which is implicitly narrowed to uint16_t. UBX payloads are length-limited to 65535 by the u2 length field, so in-spec this is latent; but a coalesced/multi-frame buffer (see UBX-05) or any payload approaching 64KB causes `offset` to wrap to a small value, producing a wrong (in-bounds-looking) read instead of the intended one — silent data corruption rather than a clean error. The signedness is fine (all unsigned) but the width is wrong for the size_t offsets the call sites compute.

**Evidence.** utils.hpp:194 `inline T buf_offset(std::vector<u1_t> * buf, u2_t offset)`; rawx.hpp:110-129 compute `size_t offset` and pass it, narrowing to u2_t each call.

**Repro.** Construct a payload where a meas_data offset exceeds 65535; the read silently aliases low memory of the same buffer.

**Suggested fix.** Change buf_offset's offset parameter to size_t and keep all call-site offsets as size_t.

**Verification.** VERIFIED the type-width mismatch exists. utils.hpp:194 is `inline T buf_offset(std::vector<u1_t> * buf, u2_t offset)` — offset is u2_t (uint16_t). The RAWX loop computes `size_t offset = 16` and `offset += 32`, then passes it to buf_offset, narrowing size_t -> uint16_t on every call. So the implicit narrowing the finding describes is real.

CORRECTION 1 (file:line): The cited path `rawx.hpp:110-129` does NOT exist. The actual call site is ublox_dgnss_node/include/ublox_dgnss_node/ubx/rxm/ubx_rxm_rawx.hpp:110 (`size_t offset = 16`), :113-126 (buf_offset calls with offset/offset+N), :129 (`offset += 32`). evidence is otherwise accurate.

CORRECTION 2 (reachability / severity is overstated): The finding's claim that "any payload approaching 64KB causes offset to wrap... silent data corruption" CANNOT occur for RXM-RAWX. num_meas is u1_t (ubx_rxm_rawx.hpp:81), max 255, so the maximum offset reached is 16 + 255*32 = 8176 — nowhere near the uint16_t boundary (65536). Additionally the constructor takes `u2_t size` (ubx_rxm_rawx.hpp:93) and all UBX payloads are length-limited to 65535 by the …

---

### USB-09 — err_count_ muting hides persistent IN errors instead of escalating; non-atomic counters shared across event callbacks

- **Severity:** P3  **Probability:** high  **Class:** wedge-recovery  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/usb.cpp:659, 689-691, 149`

**What's wrong.** err_count_ is reset to 0 on every COMPLETED IN transfer (659) and only the first 10 consecutive errors are reported (689). After 10 errors the node goes silent AND keeps resubmitting (706) with no state change — so an operator sees the stream stop with no further logs and no recovery, which is precisely the 'silent wedge' user experience. err_count_ is size_t (ush.hpp:149), incremented in callback_in on the event thread; combined with USB-01/USB-02 there is no point at which a sustained error count triggers reset/reopen. It is a policy bug: the counter is used to throttle logging rather than to drive recovery.

**Evidence.** usb.cpp:659 `err_count_ = 0;` on success; 689 `if (++err_count_ < 10)` gates the exception callback; nothing acts on err_count_ crossing a threshold (no reset/close/reopen).

**Suggested fix.** Repurpose err_count_ (or add a dedicated counter) so that crossing a threshold of consecutive non-COMPLETED IN transfers escalates to clear_halt -> reset_device -> close/reopen, and emit a throttled-but-persistent WARN so the wedge is visible. Keep logging throttled but never fully silent while errors continue.

**Verification.** Cited lines verified verbatim. usb.cpp:659 `err_count_ = 0;` on COMPLETED IN transfer; usb.cpp:689 `if (++err_count_ < 10)` gates exception_cb_fn_; usb.hpp:149 `size_t err_count_ = 0;`; resubmit at usb.cpp:706 regardless of error count. grep confirms err_count_ appears in only 3 places (decl, reset, throttle) — nothing reads it to drive recovery. The core policy observation is REAL: after 10 consecutive non-COMPLETED IN transfers (ERROR/TIMED_OUT/STALL/OVERFLOW), the exception callback is muted while resubmission continues, so logs go silent with no state change.

However two parts of the finding are wrong and lower the severity:

1) "hides recovery / wedge-recovery class" is overstated. The exception callback ublox_exception_callback (ublox_dgnss_node.cpp:1714-1718) does ONLY RCLCPP_ERROR — it triggers no reset, reopen, clear_halt, or state change. So muting at usb.cpp:689 suppresses LOGGING ONLY; there is no recovery action being hidden because none exists on this path. The only real recovery path is no_device_streak_ -> attached_=false (usb.cpp:953-955), which is on the separate h …

---

### USB-10 — Synchronous read_chars throws TimeoutException with no recovery and no caller — dead but dangerous if re-enabled

- **Severity:** P3  **Probability:** low  **Class:** wedge-recovery  **Verify confidence:** high
- **Location:** `ublox_dgnss_node/src/usb.cpp:507-527`

**What's wrong.** read_chars/read_char (507) issues a synchronous libusb_bulk_transfer with timeout_ms_=250 and throws TimeoutException on LIBUSB_ERROR_TIMEOUT (517-518) with no clear_halt/reset/reopen. It reads devh_ (515) and ep_data_in_addr_ unguarded. It is currently unused (callers commented out in node.cpp:1952-1972), so it does not cause the live wedge, but if re-enabled it would add the same no-recovery timeout behavior plus the same unlocked devh_ access as the async path. Worth flagging so a future change does not reintroduce a blocking, unrecoverable read on the executor thread.

**Evidence.** usb.cpp:514-518 `libusb_bulk_transfer(devh_, ep_data_in_addr_|LIBUSB_ENDPOINT_IN, ...); if (rc == LIBUSB_ERROR_TIMEOUT) { throw TimeoutException(); }`. Only callers are commented out: node.cpp:1952 `// len = usbc_->read_chars(...)`. No lock around devh_ read at usb.cpp:515.

**Suggested fix.** Either remove the dead sync read path, or if kept, document that it must not be called from an executor callback, guard devh_ under the state lock, and add stall/reset recovery on persistent timeout before any re-enable.

**Verification.** Verified against live source at usb.cpp:507-527. Code matches the finding exactly: read_chars issues a synchronous libusb_bulk_transfer(devh_, ep_data_in_addr_|LIBUSB_ENDPOINT_IN, ..., timeout_ms_) and throws TimeoutException on LIBUSB_ERROR_TIMEOUT (517-518) and UsbException otherwise (519-523), with no clear_halt/reset/reopen in the function itself. timeout_ms_ is confirmed = 250 (usb.cpp:44). The devh_ read at 515 is unguarded by any lock, unlike write_char (534) which takes write_mutex_ around its libusb_bulk_transfer — so the "unlocked devh_" claim holds.

Dead-code claim is correct but with one correction to the caller list the finding gave. The finding only cited the commented-out caller at node.cpp:1952. There is ALSO a live (non-commented) caller: get_polled_frame() in ublox_dgnss_node/include/ublox_dgnss_node/ubx/ubx.hpp:130. However get_polled_frame itself has zero callers anywhere (grep confirms only its definition), so read_chars is effectively dead overall — the finding's core "no live caller / does not cause the live wedge" conclusion is correct.

Minor nuance, not sev …

---
