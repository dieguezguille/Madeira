# Steam's CEG step fails with EResult 2 without system32\mui

## Symptom

A 32-bit Steam game protected by CEG (its appmanifest carries `CheckGuid`)
does not start. Madeira Dock stops with "Steam could not prepare this game's
executable for your account (code 2)". The custom-binary job ends about a
minute after Dock requests it, right after Steam's client service finishes
its work, so it is not a client timeout.

## Cause

For this step Steam's 32-bit client service (`SteamService.exe`) maps a small
module that collects the identity of a few directories. Wine relay on the
service shows it walking the subdirectories of `C:\windows` and
`C:\windows\system32` with `FindFirstFileW`, `CreateFileW(...,
FILE_FLAG_BACKUP_SEMANTICS)` and `GetFileInformationByHandle`, and at the end
reading these four:

    C:\Program Files (x86)\Steam\<a subdirectory>
    C:\windows\system32\mui
    C:\windows\Fonts
    C:\Program Files (x86)\Steam\config

`C:\windows\system32\mui` (for a 32-bit process, `C:\windows\syswow64\mui`
through the WoW64 redirection) does not exist in a Madeira prefix. Both
`CreateFileW` calls fail, the module reports failure, and Steam ends the job
with EResult 2.

On a normal Wine prefix the directory exists: `wine.inf` creates it with
`11,mui,` in `[FakeDlls]` and `[FakeDllsWow64]`, during wineboot's fake-DLL
install. Madeira never runs that install (the same reason the x86 winsxs store
is seeded by the app), and the prefix template does not carry it.

## Fix

`WineProcessBridge.m` creates the two empty directories every session, next to
the other WoW64 prefix setup: `system32\mui` always, and `syswow64\mui` when
the bundle has the i386 set. Nothing is written into them, so the prefix
matches what wine.inf would have produced. `docs/WOW64.md` lists them.

## Verification

On an iPhone (iOS 27.0.1) with Madeira 0.1.1, creating both directories by
hand was the last change needed: Steam then returned `ceg-job-result=1`, the
game process started, and the game reached its title screen. The app change
here does the same at session start; it has not been built through the full
app build here.

## Related

The same module walks `C:\windows` and `system32` roughly a thousand times per
pass, re-enumerating the directory for every step, and makes three passes.
In a Wine prefix, which has few subdirectories, it keeps landing on the last
one. On the iPhone this takes about 55 seconds, mostly in `stat`, `lstat` and
`listxattr` from `NtQueryDirectoryFile`. The client waited for it in the
successful run, but faster directory enumeration would leave more margin.
