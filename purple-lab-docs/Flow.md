# Sliver Execution Methods — Flow

**Entry:** Operator CLI → `client/command/exec/*` → gRPC → implant handlers → `taskrunner/task_windows.go`
**Artifact summary:** 8 execution paths covering process spawn, shellcode injection, .NET assembly, reflective DLL, BOF, WASM, interactive shell, and process migration.

---

## Method 1 — `execute` (cross-platform process spawn)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 1.1 | operator sends `ExecuteReq` protobuf over gRPC tunnel | `[no-artifact]` → #1.2 | — | gRPC stream to server over mTLS/HTTP |
| 1.2 | server forwards task to implant over C2 transport | `[no-artifact]` → #1.3 | — | encrypted C2 channel |
| 1.3 | implant spawns child process via `exec.Command` | child process [process] → #1.4 | — | parent = implant process; no PPID manipulation |
| 1.4 | implant captures child stdout/stderr into buffer | output bytes [memory] → #1.5 | — | pipe attached to child; synchronous `cmd.Run()` |
| 1.5 | implant returns output bytes to operator via C2 | `[no-artifact]` | — | response protobuf over same transport |

**Windows-only variant (`ExecuteWindows`):**

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 1W.1 | implant sets `SysProcAttr.Token` to current impersonation token | identity [identity] → #1W.2 | — | token set before `CreateProcess`; not via `SetThreadToken` |
| 1W.2 | implant spoofs parent PID via `spoof.SpoofParent` before spawn | child process under spoofed parent [process] → #1W.3 | — | `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS`; visible in process tree |
| 1W.3 | implant spawns child with `HideWindow = true` | child process [process] → #1W.4 | — | `CREATE_NO_WINDOW`; no console window created |
| 1W.4 | implant captures or detaches child output | output bytes / `[no-artifact]` [memory] | — | with `--background`: no pipe, no wait |

---

## Method 2 — `execute-shellcode` (shellcode injection)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 2.1 | client converts PE/DLL to shellcode via Donut | shellcode bytes [memory] → #2.2 | — | optional; Donut runs on operator machine |
| 2.2 | implant patches AMSI in-process via byte overwrite (`patchAmsi`) | modified AMSI memory [memory] → #2.4 | — | `AmsiScanBuffer` prologue overwritten; no file write |
| 2.3 | implant patches ETW in-process via byte overwrite (`patchEtw`) | modified ETW memory [memory] → #2.4 | — | `EtwEventWrite` prologue overwritten |
| 2.4 | implant allocates RWX region and writes shellcode (`LocalTask`) | RWX private allocation [memory] → #2.5 | — | private commit, no file backing, RWX on allocation |
| 2.5 | implant creates thread at shellcode base address | executing thread [process] → terminal | — | start address in private RWX region outside any image |

**Remote inject variant (`RemoteTask`):**

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 2R.1 | implant opens handle to target process | cross-process handle [process] → #2R.2 | — | `OpenProcess` with PROCESS_VM_WRITE + CREATE_THREAD |
| 2R.2 | implant allocates RWX memory in target process (`VirtualAllocEx`) | remote RWX region [memory] → #2R.3 | — | private commit in remote process |
| 2R.3 | implant writes shellcode into remote region (`WriteProcessMemory`) | shellcode bytes in remote [memory] → #2R.4 | — | cross-process memory write |
| 2R.4 | implant creates remote thread at shellcode address (`CreateRemoteThread`) | remote thread [process] → terminal | — | thread start address in remote private region |

---

## Method 3 — `execute-assembly` (in-memory .NET)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 3.1 | implant patches AMSI in-process (`patchAmsi`) | modified AMSI memory [memory] → #3.3 | — | same as #2.2 |
| 3.2 | implant patches ETW in-process (`patchEtw`) | modified ETW memory [memory] → #3.3 | — | same as #2.3 |
| 3.3 | implant loads CLR runtime into process via `CorBindToRuntimeEx` | CLR loaded in implant process [memory] → #3.4 | — | CLR loaded into non-.NET host process |
| 3.4 | implant invokes assembly entry point in-process (`InProcExecuteAssembly`) | executing managed code [process] → #3.5 | — | assembly runs in implant process address space; no disk write |
| 3.5 | implant captures assembly stdout/stderr and returns to operator | output bytes [memory] | — | pipe capture from managed AppDomain |

---

## Method 4 — `spawndll` / `sideload` (reflective DLL)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 4.1 | implant spawns sacrificial process (e.g. `notepad.exe`) | child process [process] → #4.2 | — | process created for injection target; not for execution |
| 4.2 | implant allocates RWX region in sacrificial process | remote RWX region [memory] → #4.3 | — | private commit in target |
| 4.3 | implant writes DLL bytes into remote region | DLL bytes in remote [memory] → #4.4 | — | cross-process memory write |
| 4.4 | implant creates thread to execute DLL entry point reflectively | remote thread at DLL entry [process] → #4.5 | — | reflective loader; no `LoadLibrary` call; DLL not in PEB module list |
| 4.5 | implant captures stdout pipe from sacrificial process | output bytes [memory] | — | pipe from child stdout |

**sideload variant (`extension_windows.go`):**

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 4S.1 | implant loads extension DLL via `LoadLibrary` | DLL loaded in implant process [memory] → #4S.2 | — | DLL appears in PEB module list; loaded in implant process |
| 4S.2 | implant resolves exported function via `GetProcAddress` | function pointer [memory] → #4S.3 | — | `[no-artifact]` link — API resolution |
| 4S.3 | implant calls extension entry point | extension code executing [process] | — | execution in implant process context |

---

## Method 5 — `execute-bof` (Beacon Object File)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 5.1 | client marshals typed args to Cobalt Strike BOF wire format (`BOFArgsBuffer`) | arg bytes [memory] → #5.2 | — | format compatible with CS BOF loaders |
| 5.2 | implant loads BOF object file into implant process memory | BOF object in memory [memory] → #5.3 | — | raw COFF object; not a PE/DLL; no disk write |
| 5.3 | implant resolves Beacon API symbols for BOF | `[no-artifact]` → #5.4 | — | Beacon API shimmed in implant; BOF sees CS-compatible API |
| 5.4 | implant executes BOF in-process (`executeBOFWithLoader`) | BOF output bytes [memory] → #5.5 | — | executes in implant thread; no new process |
| 5.5 | implant collects typed BOF output and returns to operator | output bytes [memory] | — | typed output: strings, ints, binary blobs |

---

## Method 6 — `wasm` (WebAssembly extension)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 6.1 | operator uploads WASM module bytes to implant extension registry | WASM module in memory [memory] → #6.2 | — | module stored in-process registry; no disk write |
| 6.2 | implant initializes Wazero runtime for module execution | WASM runtime instance [memory] → #6.3 | — | Wazero runs in implant process; sandboxed |
| 6.3 | implant executes WASM module function (`ExecWasmExtensionHandler`) | module output [memory] → #6.4 | — | execution sandboxed in Wazero; host syscalls via import functions |
| 6.4 | implant pipes WASM stdin/stdout through C2 tunnel (interactive mode) | bidirectional byte stream [network] | — | tunnel-bound I/O; same path as shell tunnel |

---

## Method 7 — `shell` (interactive shell)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 7.1 | client creates bidirectional tunnel via `CreateTunnel` RPC | tunnel ID [network] → #7.2 | — | tunnel registered on server; no implant contact yet |
| 7.2 | client sends `ShellReq` with tunnel ID and PTY dimensions | `[no-artifact]` → #7.3 | — | request routed over C2 transport to implant |
| 7.3 | implant resolves shell path (`GetSystemShellPath`) | `[no-artifact]` → #7.4 | — | Windows: tries `powershell.exe` first, falls back to `cmd.exe` |
| 7.4 | implant spawns shell process with pipes (`pipedShell`) | child process with stdin/stdout pipe [process] → #7.5 | — | Windows: `SysProcAttr.Token = priv.CurrentToken`, `HideWindow = true` |
| 7.5 | implant binds shell stdin/stdout to C2 tunnel (`NewTunnel`) | active bidirectional tunnel [network] → #7.6 | — | `io.Copy(tunnelWriter, shellStdout)` goroutine |
| 7.6 | implant forwards operator keystrokes to shell stdin over tunnel | shell input bytes [network] → terminal | — | byte stream; operator escape byte `0x1d` intercepted client-side only |

**PTY variant (Linux/macOS only):**

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 7P.1 | implant opens PTY master/slave pair (`pty.Open` via `/dev/ptmx`) | PTY device pair [process] → #7P.2 | — | `Setctty + Setsid`; shell gets controlling terminal |
| 7P.2 | implant sets PTY window size (`pty.Setsize`) | `[no-artifact]` → #7P.3 | — | rows/cols from operator terminal |
| 7P.3 | implant attaches shell process to slave PTY | child process with PTY [process] → tunnel | — | stdout/stdin/stderr all to ttyFile |

---

## Method 8 — `migrate` (process injection + session re-establish)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| 8.1 | implant opens handle to target process | cross-process handle [process] → #8.2 | — | `OpenProcess` with inject permissions |
| 8.2 | implant allocates RWX region in target process | remote RWX region [memory] → #8.3 | — | private commit in target |
| 8.3 | implant writes implant shellcode into remote region | implant shellcode in remote [memory] → #8.4 | — | full implant payload injected |
| 8.4 | implant creates remote thread to execute implant shellcode | remote thread [process] → #8.5 | — | start address in remote private RWX region |
| 8.5 | injected implant initiates new C2 connection from target process | new C2 session [network] | — | target process now has outbound C2 connection |

---

## Composite Paths

### Path A — Tiered shellcode execution (session)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| A.1 | implant patches AMSI and ETW in-process | modified ASM memory [memory] → #A.2 | — | prerequisite for all in-proc execution below |
| A.2 | implant self-injects shellcode via `LocalTask` | RWX alloc + thread [memory, process] → terminal | — | see Method 2 detail |
| A.3 | implant loads CLR and executes .NET assembly in-process | CLR + managed execution [memory, process] → terminal | — | see Method 3 detail; can follow A.2 in same session |
| A.4 | implant loads and executes BOF object in-process | BOF execution [memory] → terminal | — | see Method 5 detail; no new process at any step |

### Path B — Lateral / privilege (Windows)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| B.1 | implant creates impersonation token via `MakeToken` or `ImpersonateLoggedOnUser` | impersonation token [identity] → #B.2 | — | `priv.CurrentToken` set for subsequent operations |
| B.2 | implant spawns process under impersonation token with PPID spoof | child process under stolen identity and spoofed parent [process, identity] → #B.3 | — | `UseToken=true`, `PPid` set; both attributes set before `CreateProcess` |
| B.3 | implant injects implant shellcode into privileged process and re-establishes session | new C2 session from privileged process [process, network] | — | see Method 8 detail |

---

## Evasion Methods

### E1 — AMSI / ETW In-Memory Patch (runtime)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| E1.1 | implant resolves `AmsiScanBuffer` address in loaded `amsi.dll` | function pointer [memory] → #E1.2 | — | `[no-artifact]`; resolved from loaded module in PEB |
| E1.2 | implant marks AMSI `.text` region RWX via `VirtualProtect` | RWX page on AMSI image region [memory] → #E1.3 | — | page-protect change on loaded image; backed by `amsi.dll` on disk |
| E1.3 | implant overwrites `AmsiScanBuffer` prologue bytes in-memory | modified AMSI scan function [memory] → terminal | — | AMSI scans return clean for process lifetime; no disk change |
| E1.4 | implant resolves `EtwEventWrite` address in loaded `ntdll.dll` | function pointer [memory] → #E1.5 | — | `[no-artifact]`; same process as E1.1 |
| E1.5 | implant marks ETW function region RWX via `VirtualProtect` | RWX page on ntdll image region [memory] → #E1.6 | — | page-protect change on `ntdll.dll`-backed page |
| E1.6 | implant overwrites `EtwEventWrite` prologue bytes in-memory | silenced ETW provider [memory] → terminal | — | ETW events from process suppressed; no disk change |

### E2 — RefreshPE (EDR hook removal)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| E2.1 | implant opens DLL file from disk (`pe.Open`) | file read handle [file] → #E2.2 | — | reads clean copy of target DLL (e.g. `ntdll.dll`, `kernel32.dll`) |
| E2.2 | implant extracts `.text` section bytes from clean disk copy | clean `.text` bytes [memory] → #E2.4 | — | `pe.Section(".text").Data()` |
| E2.3 | implant loads in-memory DLL via `LoadDLL` to get base address | DLL handle [memory] → #E2.4 | — | `[no-artifact]`; handle to already-loaded module |
| E2.4 | implant marks in-memory `.text` region RWX (`VirtualProtect`) | RWX page on live DLL [memory] → #E2.5 | — | EDR hooks sit in this region |
| E2.5 | implant copies clean disk bytes over live `.text` region byte-by-byte | unhooked `.text` region [memory] → #E2.6 | — | overwrites EDR inline hooks; `unsafe.Pointer` loop |
| E2.6 | implant restores original page permissions (`VirtualProtect`) | restored permission [memory] → terminal | — | page returns to original protect; no disk artifact |

### E3 — PPID Spoofing (process tree disguise)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| E3.1 | implant opens handle to target parent process (`OpenProcess`) | parent process handle [process] → #E3.2 | — | `PROCESS_CREATE_PROCESS \| PROCESS_DUP_HANDLE \| PROCESS_QUERY_INFORMATION` |
| E3.2 | implant sets `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS` on new process | child with spoofed parent [process] → terminal | — | child appears under target parent in process tree; actual creator is implant |

### E4 — Implant Build-Time Obfuscation (pre-deploy)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| E4.1 | server compiles implant via Garble (`ObfuscateSymbols=true`) | binary with no debug symbols, randomized identifiers [file] → #E4.2 | — | strips DWARF, build ID; randomizes func/type names; `-s -w` implicit |
| E4.2 | server applies PE metadata spoof from donor binary (`spoof.go`) | implant PE with cloned version info, resources, export dir [file] → terminal | — | `VersionInfo`, `IconGroup`, export table match a legitimate binary |

### E5 — Shellcode Encoding (pre-deploy)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| E5.1 | server XOR-encodes shellcode with static or dynamic key | XOR-encoded shellcode blob [file] → terminal | — | `XorEncoder` / `XorDynamicEncoder`; bad-char aware key selection |
| E5.2 | server encodes shellcode via Shikata Ga Nai (`sgn.EncodeShellcode`) | SGN-encoded polymorphic shellcode [file] → terminal | — | polymorphic XOR; self-decoding stub prepended; signature changes per encode |

### E6 — Traffic Encoding (runtime C2)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| E6.1 | implant loads WASM traffic encoder module at startup | encoder module in memory [memory] → #E6.2 | — | module selected by C2 profile config; `TrafficEncodersEnabled` build flag |
| E6.2 | implant applies encoder pipeline to outbound C2 data before send | transformed bytes over wire [network] → terminal | — | custom byte transforms on top of mTLS/HTTP; visible as unusual byte distributions |

### E7 — Beacon Jitter (timing evasion)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| E7.1 | beacon implant computes next sleep as `Interval + rand(Jitter)` | sleep duration [memory] → #E7.2 | — | `[no-artifact]`; pure computation before sleep |
| E7.2 | beacon sleeps for computed duration before next C2 checkin | delayed checkin [network] → terminal | — | irregular interval breaks fixed-period beacon detection |

### E8 — Execution Limits / Kill-Switch (pre-exec)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| E8.1 | implant calls `IsDebuggerPresent` and exits if debugger detected | process exit [process] → terminal | — | `PlatformLimits()`; triggers on non-debug builds only |
| E8.2 | implant checks hostname against baked-in value and exits if mismatch | process exit [process] → terminal | — | `os.Hostname()` compare; `LimitHostname` build config |
| E8.3 | implant checks username against baked-in value and exits if mismatch | process exit [process] → terminal | — | `user.Current()`; Windows strips domain prefix |
| E8.4 | implant checks domain-joined status via `NetGetJoinInformation` and exits if not joined | process exit [process] → terminal | — | `LimitDomainJoined`; implant refuses to run on non-domain host |
| E8.5 | implant checks system datetime against expiry and exits if past | process exit [process] → terminal | — | RFC3339 compare; `LimitDatetime`; dead-man kill-switch |
| E8.6 | implant checks file existence and exits if marker file absent | process exit [process] → terminal | — | `os.Stat(LimitFileExists)`; sandbox marker file check |

---

### Path C — Interactive recon (Windows session)

| # | Behavior (`actor action artifact`) | Artifact [class] → consumed by | Tactic / TID | Context (baseline) |
|---|---|---|---|---|
| C.1 | implant spawns PowerShell with hidden window and current token | child `powershell.exe` process [process] → #C.2 | — | `HideWindow=true`, `Token=priv.CurrentToken` |
| C.2 | implant binds PowerShell stdin/stdout to bidirectional C2 tunnel | active tunnel [network] → #C.3 | — | piped shell; operator keystrokes forwarded |
| C.3 | operator detaches shell via `0x1d` escape byte | `[no-artifact]` → #C.4 | — | escape intercepted client-side; shell process stays alive on target |
| C.4 | implant spawns child process for discrete discovery command | child process + output [process, memory] → terminal | — | `execute -o cmd.exe /c <cmd>`; separate from live shell |
