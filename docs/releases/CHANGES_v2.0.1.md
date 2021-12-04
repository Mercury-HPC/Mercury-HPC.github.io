---
title: Release Notes v2.0.1
---

## Summary

This version brings a few bug fixes and updates to our v2.0.0 release.

## New features

- Improve logging system and add `HG_LOG_SUBSYS` environment variable that can be used
in combination with `HG_LOG_LEVEL` to select log sub-systems.
    * Add `min_debug` log level to keep debug traces. Traces get printed when an error occurs.
    * __[HG]__ Add `HG_Set_log_subsys()`.
- __[NA]__ Add support for message size hints though max_unexpected_size and max_expected_size hints. Supported with OFI, BMI and MPI plugins.
- __[NA BMI/SM/OFI]__ Support sends to self address.
- __[HG/NA]__ Add `HG_HOSTUNREACH`/`NA_HOSTUNREACH` error codes.

## Bug fixes

- __[HG]__
    - Add missing check for NULL addr passed to `HG_Forward()`.
    - Remove unnecessary `spinwait()` and track handle in completion queue.
    - Fix handle refcount if `HG_Respond()` fails.
    - Remove race in `HG_Trigger()` optimization that was skipping signaling.
- __[HG bulk]__
    - Prevent virtual handle data to be sent eagerly.
    - Ensure underlying error codes from NA are returned back to user.
- __[HG util]__
    - Fix timeout passed to `pthread_cond_timedwait()` when
    `CLOCK_MONOTONIC_COARSE` is used.
    - Remove check for `STDERR_FILENO`.
    - Add best-effort C++ compatibility for atomics.
- __[NA]__
    - Ensure completion callback is called after OP ID is fully released.
- __[NA BMI]__
    - Rework and simplify NA BMI code and remove extra allocations.
- __[NA SM]__
    - Prevent potential race on bulk handle that was freed.
    - Fix release of invalid addresses.
    - Prevent race in address resolution.
- __[NA OFI]__
    - Allow libfabric to return canceled operations.
    - Yield to other threads when using PSM2.
    - Return and convert OFI error codes back to upper layers.
    - Ensure selected domain matches address format.
    - Prevent tcp protocol to be used on macOS.
    - Fix potential memory leak in `na_ofi_provider_check()`.
    - Add addr pool to prevent addr allocation on unexpected recv.

## :warning: Known Issues

- __[NA OFI]__
    - [tcp/verbs;ofi_rxm] Using more than 256 peers requires `FI_UNIVERSE_SIZE` to be set.
    - [tcp;ofi_rxm] Remains unstable, use `sockets` as a fallback in case of issues.
