# 64-bit processes cannot create threads while a 32-bit process is alive

## Symptom

As soon as any 32-bit process runs in the session, a 64-bit process that
needs a new block of TEBs cannot create the thread that needed it. The case
that exposed it: Steam starts its 32-bit client service through
`services.exe` (aarch64). When the service connects, `services.exe` tries to
start an RPC I/O thread, the thread creation fails with `STATUS_NO_MEMORY`,
and the service's RPC calls to `\pipe\svcctl` fail with
`RPC_S_CALL_FAILED` (1726).

## Cause

`virtual_alloc_teb` (`build/ntdll-unix/virtual_ios.c`) allocates a new TEB
block with `zbits = user_space_wow_limit`, the guest ceiling that
`allocate_virtual_memory` turns into `[B, B + limit]` for a process with a
32-bit guest window.

`user_space_wow_limit` is a session global (ml1270, the same reason as
`ios_section_zero_bits`): it is nonzero whenever any 32-bit process is alive.
A 64-bit process has no guest window, so for it that value is an impossible
sub-4-GB host range. The allocation fails, and with it the thread.

## Fix

In `virtual_alloc_teb`, after the existing WoW64 fallback, clear the ceiling
for a 64-bit process:

    if (!wow) zbits = 0;

32-bit processes keep the existing behaviour, including the conservative
2-GB answer for a thread created before `init_peb` publishes the limit.

## Verification

Tested on an iPhone (iOS 27.0.1) with Madeira 0.1.1 through an equivalent
binary patch of the shipped `Madeira.debug.dylib`. With it, `services.exe`
creates its RPC threads while the 32-bit Steam service runs, and the
service's calls to the service manager succeed. The source change has not
been built through the full app build here.
