# KrakenSDR DOA Codebase Bug Analysis Report

**Date:** 2025-12-28
**Analyzed by:** Claude Code

## Executive Summary

A comprehensive code review of the KrakenSDR DOA codebase identified **19 bugs** across 6 categories. The most critical issues involve resource leaks, thread safety problems, and logic errors that could cause incorrect direction-of-arrival calculations.

---

## Critical Bugs

### 1. Resource Leak: File Descriptors Never Closed

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:98, 229`

**Problem:**
```python
# Line 98 - DOA_res_fd opened but never closed
self.DOA_res_fd = open(doa_res_file_path, "w+")

# Line 229 - data_record_fd opened but never closed
self.data_record_fd = open(data_recording_file_path, "a+")
```

**Impact:** The `SignalProcessor` class opens file handles in `__init__` but has no destructor or cleanup method to close them. The thread runs indefinitely in a `while True` loop, causing file descriptor leaks over time.

**Recommended Fix:** Implement a `close()` method or use context managers.

---

### 2. Missing Return Value in get_iq_online()

**Location:** `_sdr/_receiver/kraken_sdr_receiver.py:180-231`

**Problem:**
```python
def get_iq_online(self):
    if self.data_interface == "eth":
        self.socket_inst.sendall(str.encode("IQDownload"))
        self.iq_samples = self.receive_iq_frame()
    elif self.data_interface == "shmem":
        # ... processes data
        self.in_shmem_iface.send_ctr_buff_ready(active_buff_index)
    # BUG: No return statement for the shmem path - returns None implicitly
```

**Impact:** The caller expects a return value to check for failure:
```python
get_iq_failed = self.module_receiver.get_iq_online()  # Line 363
```
In `shmem` mode, this always returns `None`, which is falsy, potentially masking errors.

**Recommended Fix:** Add explicit `return 0` at the end of the `shmem` branch to indicate success.

---

### 3. Always-True Condition in gen_scanning_vectors()

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:1631`

**Problem:**
```python
elif "ULA":  # BUG: This is always True!
    x = np.zeros(M)
    y = -np.arange(M) * DOA_inter_elem_space
```

**Impact:** This condition is always `True` because `"ULA"` is a non-empty string. The code should compare against the `type` parameter: `elif type == "ULA":`. This means any array type that's not "UCA" will be treated as ULA, even if it's "Custom".

**Recommended Fix:** Change to `elif type == "ULA":`.

---

## Thread Safety Issues

### 4. Race Condition: Shared State Modified Without Locks

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py` (multiple locations)

**Problem:** The `SignalProcessor` thread modifies many shared state variables that are read by the web interface callbacks without any synchronization:
- `self.DOA`, `self.spectrum`, `self.vfo_freq`, `self.vfo_squelch`
- `self.doa_max_list`, `self.theta_0_list`, `self.confidence_list`

**Impact:** Potential for reading partially-written data or inconsistent state, leading to garbled display or crashes.

**Recommended Fix:** Use `threading.Lock` for critical sections or use thread-safe data structures like `queue.Queue`.

---

### 5. Timer Race Condition

**Location:** `_ui/_web_interface/utils.py:228-229, 241-242`

**Problem:**
```python
web_interface.dsp_timer = Timer(0.01, fetch_dsp_data, args=(...))
web_interface.dsp_timer.start()
```

When disconnecting:
```python
web_interface.dsp_timer.cancel()  # callbacks/main.py:53
```

**Impact:** The timer might fire between `cancel()` call and the old timer completing, causing `AttributeError` or accessing freed resources.

**Recommended Fix:** Use a flag variable to prevent rescheduling instead of relying solely on `cancel()`.

---

## Logic Errors

### 6. Wrong Variable Name Check

**Location:** `_ui/_web_interface/callbacks/main.py:633-634`

**Problem:**
```python
if is_int(compass_offset):  # BUG: Should check array_offset
    web_interface.module_signal_processor.array_offset = array_offset
```

**Impact:** `array_offset` is set even if it's not a valid integer, since the wrong variable is being validated.

**Recommended Fix:** Change to `if is_int(array_offset):`.

---

### 7. Division by Zero Risk

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:744-745`

**Problem:**
```python
daq_cpi = int(
    self.module_receiver.iq_header.cpi_length * 1000 / self.module_receiver.iq_header.sampling_freq
)
```

**Impact:** If `sampling_freq` is zero (e.g., during initialization or error state), this causes a `ZeroDivisionError`.

**Recommended Fix:** Add a guard: `if self.module_receiver.iq_header.sampling_freq > 0:`.

---

### 8. Index Out of Bounds Risk

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:810-812`

**Problem:**
```python
self.number_of_correlated_sources[0],
self.snrs[0],
```

**Impact:** These lists are cleared at lines 499-500 and only appended during active VFO processing. If no VFOs pass the squelch check, the lists remain empty, causing `IndexError`.

**Recommended Fix:** Add bounds checking or use default values when lists are empty.

---

### 9. Incorrect Index Check in Custom Array

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:1654-1655`

**Problem:**
```python
for i in range(len(custom_x)):
    if i > M:  # BUG: Should be i >= M
        break
```

**Impact:** This allows writing to `x[M]` if `len(custom_x) > M`, causing an index out of bounds error.

**Recommended Fix:** Change to `if i >= M:`.

---

### 10. Wrong Variable Check for Y Array

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:1665`

**Problem:**
```python
if custom_x[i] == "":  # BUG: Should check custom_y[i]
    y[i] = 0
```

**Impact:** The check uses `custom_x[i]` but should use `custom_y[i]` when setting `y[i]`, causing incorrect array values.

**Recommended Fix:** Change to `if custom_y[i] == "":`.

---

## Error Handling Issues

### 11. Bare Exception Catch

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:733-735`

**Problem:**
```python
except Exception:
    self.logger.error(traceback.format_exc())
    self.data_ready = False
```

**Impact:** This catches all exceptions including `KeyboardInterrupt` and `SystemExit`, making it difficult to stop the application gracefully.

**Recommended Fix:** Catch specific exceptions or exclude `BaseException` subclasses.

---

### 12. Return Type Inconsistency

**Location:** `_sdr/_receiver/kraken_sdr_receiver.py:284-285`

**Problem:**
```python
else:
    return 0
```

**Impact:** When `incoming_payload_size` is 0, the function returns `0` (integer) instead of an empty numpy array, causing type inconsistency with other code paths.

**Recommended Fix:** Return `np.empty(0)` or `None` with proper handling.

---

### 13. Unusual Exception Handling

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:1066`

**Problem:**
```python
except (gpsd.NoFixError, UserWarning, ValueError, BrokenPipeError):
```

**Impact:** Catching `UserWarning` as an exception is unusual. `UserWarning` is typically used with `warnings.warn()`, not raised as an exception.

**Recommended Fix:** Remove `UserWarning` from the exception tuple unless the gpsd library actually raises it.

---

## Memory Issues

### 14. Unbounded Cache Growth

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py` (multiple functions)

**Problem:** Multiple `@lru_cache` decorated functions cache numpy arrays with varying parameters:
- `get_fir`, `get_exponential`, `shift_filter`, `T`, `gen_scanning_vectors`

**Impact:** If parameters (especially frequency) change frequently, the caches grow until `maxsize` is hit. Each cached entry holds large numpy arrays in memory.

**Recommended Fix:** Consider clearing caches on frequency changes or use smaller `maxsize` values.

---

### 15. Division by Zero in audible()

**Location:** `_sdr/_signal_processing/signal_utils.py:19`

**Problem:**
```python
signal = np.int16(signal / np.max(np.abs(signal)) * 32767)
```

**Impact:** If `np.max(np.abs(signal))` is zero (silence), this causes division by zero.

**Recommended Fix:** Add guard: `max_val = np.max(np.abs(signal)); if max_val > 0: ...`.

---

## Minor Bugs

### 16. Typo in Method Name

**Location:** `_sdr/_receiver/shmemIface.py:106, 190`

**Problem:**
```python
def destory_sm_buffer(self):  # Should be "destroy_sm_buffer"
```

**Impact:** Not a runtime issue since typo is used consistently, but indicates code quality issues.

**Recommended Fix:** Rename to `destroy_sm_buffer`.

---

### 17. sys.getsizeof() Misuse

**Location:** `_sdr/_receiver/kraken_sdr_receiver.py:400`

**Problem:**
```python
msg_bytes = cmd + bytearray(128 - sys.getsizeof(cmd))
```

**Impact:** `sys.getsizeof()` returns memory size including Python object overhead (~49 bytes for small bytes objects), not the byte length. This likely creates incorrect message padding.

**Recommended Fix:** Use `len(cmd)` instead of `sys.getsizeof(cmd)`.

---

### 18. Commented Typos

**Location:** `_sdr/_receiver/shmemIface.py:56, 63`

**Problem:**
```python
# shmem_A.unkink()  # Commented typo - should be "unlink"
```

**Impact:** None (commented out), but indicates potential historical bugs.

---

### 19. Inconsistent vfo_demod_modes Usage

**Location:** `_sdr/_signal_processing/kraken_sdr_signal_processor.py:633`

**Problem:**
```python
if self.vfo_demod_modes[i] or self.vfo_iq_enabled[i]:
```

**Impact:** `self.vfo_demod_modes[i]` returns a string (e.g., "None", "FM"), so this check is always truthy unless the string is empty. It should likely check for a specific mode like `!= "None"`.

---

## Summary Table

| Category | Count | Severity |
|----------|-------|----------|
| Critical Bugs | 3 | High |
| Thread Safety | 2 | High |
| Logic Errors | 5 | Medium |
| Error Handling | 3 | Medium |
| Memory Issues | 2 | Low |
| Minor Bugs | 4 | Low |
| **Total** | **19** | - |

---

## Recommendations

1. **Immediate Priority:**
   - Fix the `elif "ULA":` always-true condition (#3)
   - Add return values to `get_iq_online()` for shmem mode (#2)
   - Fix the array offset validation (#6)

2. **Short-term:**
   - Implement proper resource cleanup for file handles (#1)
   - Add thread synchronization for shared state (#4)
   - Add bounds checking for list access (#8)

3. **Long-term:**
   - Refactor exception handling to be more specific (#11)
   - Add unit tests for edge cases like zero division
   - Consider using dataclasses or typed attributes for configuration
