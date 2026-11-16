# Sliver Execution Methods — Purple Lab Reference

> Tóm tắt các phương án thực thi qua Sliver C2 agent, kèm behavior chain và ATT&CK mapping.
> Session-only trừ khi ghi chú beacon.

---

## 1. `execute` — Process Spawn

**Command:**
```
execute [-o] [--background] [--ppid <pid>] [--token] [--hidden] [--env K=V] <path> [args]
```

**Behavior chain:**
1. Client build `ExecuteReq` / `ExecuteWindowsReq` protobuf
2. gRPC → `server/rpc/rpc-execute.go`
3. C2 transport → implant `executeHandler` / `executeWindowsHandler`
4. Implant: `exec.Command(path, args)` + optional `SysProcAttr`
5. Output: capture vào buffer → protobuf response → client terminal

**Windows-only flags:**
- `--ppid <pid>` → `spoof.SpoofParent()` — parent process spoofing (T1134.004)
- `--token` → `SysProcAttr.Token = priv.CurrentToken` — chạy dưới impersonation token (T1134.001)
- `--hidden` → `HideWindow = true`

**Không có `--output`:** dùng `startExecuteChild()` → no stdin/stdout pipe với implant → ít artifact hơn.

**ATT&CK:** T1106 (Native API), T1134.001/004 (khi dùng token/ppid)

---

## 2. `execute-shellcode` — Shellcode Injection

**Command:**
```
execute-shellcode [--pid <pid>] [--inject] <shellcode.bin>
```

**Behavior chain:**
1. Client: optional Donut conversion (PE/DLL → shellcode)
2. gRPC → `server/rpc/rpc-tasks.go`
3. Implant `task_windows.go`:
   - **Remote inject (--pid):** `RemoteTask` → `VirtualAllocEx` + `WriteProcessMemory` + `CreateRemoteThread`
   - **Local (self-inject):** `LocalTask` → `VirtualAlloc` + thread creation in-process
4. Trước khi thực thi: `patchAmsi()` + `patchEtw()` — patch in-memory

**ATT&CK:** T1055.001 (remote), T1620 (local/reflective), T1562.001 (AMSI/ETW patch)

---

## 3. `execute-assembly` — In-Memory .NET

**Command:**
```
execute-assembly [--process <sacrifice>] <assembly.exe> [args]
```

**Behavior chain:**
1. Client gửi assembly bytes qua gRPC
2. Implant `executeAssemblyHandler` → `taskrunner/task_windows.go`
3. `patchAmsi()` + `patchEtw()`
4. `ExecuteAssembly` → load CLR in-process → invoke assembly entry point
5. Output capture → trả về client

Không ghi disk. CLR load vào memory của implant process.

**ATT&CK:** T1620 (Reflective Code Loading), T1059.001 (nếu dùng PowerShell assembly)

---

## 4. `sideload` / `spawndll` — Reflective DLL

**Commands:**
```
sideload <lib.dll> [args]
spawndll [--pid <pid>] <lib.dll> [args]
```

**Behavior chain (spawndll):**
1. Client gửi DLL bytes
2. `spawnDllHandler` → `taskrunner/task_windows.go:SpawnDll`
3. Spawn sacrificial process
4. Inject DLL vào sacrificial process → gọi entry point reflectively
5. Output capture từ stdout pipe của sacrificial process

**sideload:** dùng `LoadLibrary`/`GetProcAddress` trong extension loader (`extension/extension_windows.go`)

**ATT&CK:** T1055.001 (DLL injection), T1055.002 (reflective DLL)

---

## 5. `execute-bof` — Beacon Object File

**Command:**
```
execute-bof <bof.o> [args]
```

**Behavior chain:**
1. Client marshal typed args → `BOFArgsBuffer` (Cobalt Strike wire format)
2. gRPC → implant `executeBOFWithLoader` (`handlers/handlers_bof.go`)
3. Load BOF object file vào memory của implant
4. Resolve Beacon API symbols
5. Execute BOF → collect typed output → trả về client

Chạy trong memory của implant process, không spawn process mới.

**ATT&CK:** T1620 (Reflective Code Loading)

---

## 6. `wasm` — WebAssembly Extension

**Command:**
```
wasm register <module.wasm>
wasm [--interactive] run <module-name> [args]
```

**Behavior chain:**
1. Register: upload WASM module → lưu vào extension registry của implant
2. Run: `ExecWasmExtensionHandler` → Wazero runtime execute module
3. Interactive mode: stdin/stdout piped qua tunnel (tương tự shell)
4. Non-interactive: capture output → trả về

Portable, cross-platform. Module được execute trong sandbox Wazero.

**ATT&CK:** T1620

---

## 7. `shell` — Interactive Shell

**Command:**
```
shell [--shell-path <path>] [--no-pty]
shell ls | attach <id> | kill [--force] <id>
```

**Behavior chain:**
1. Client: `CreateTunnel()` → server tạo bidirectional tunnel ID
2. `Rpc.Shell(ShellReq)` → gửi tunnelID + path + PTY size
3. Implant `ShellReqHandler`: `StartInteractive()` → spawn shell process
4. `NewTunnel()` bind shell stdin/stdout vào C2 tunnel
5. I/O forward: `io.Copy(tunnelWriter, shellStdout)` (goroutine)
6. Operator input: scan stdin cho `0x1d` (Ctrl-]) → escape/detach

**Platform behavior:**
- Linux/macOS: PTY (`ptyShell`) — tab completion, job control, resize
- Windows: piped only — spawn PowerShell → fallback cmd.exe; `HideWindow=true`; dùng `priv.CurrentToken`

**Escape keys:**
- `Ctrl-]` (`0x1d`) → **detach** (shell vẫn live, `shell attach <id>` để re-attach)
- `exit` / `logout` / `Ctrl-D` → **close** (gửi lệnh rồi đóng tunnel)

**Session only** — không hoạt động với beacon.

**ATT&CK:** T1059.001 (PowerShell), T1059.003 (cmd), T1071.001 (tunnel over HTTP/mTLS)

---

## 8. `migrate` — Process Migration

**Command:**
```
migrate --pid <pid>
```

**Behavior chain:**
1. `migrateHandler` inject shellcode implant vào process đích
2. Shellcode khởi tạo C2 connection mới từ process đó
3. Session mới được tạo; session cũ vẫn tồn tại cho đến khi kill

Dùng để di chuyển sang process có privilege cao hơn hoặc ít bị monitor.

**ATT&CK:** T1055.001, T1134 (nếu inject vào privileged process)

---

---

## Evasion Methods

### E1 — AMSI / ETW In-Memory Patch

Vô hiệu hóa runtime scanning trước khi thực thi shellcode / assembly.

- `patchAmsi()` → resolve `AmsiScanBuffer` trong `amsi.dll` → `VirtualProtect(RWX)` → overwrite prologue bytes → restore permissions
- `patchEtw()` → resolve `EtwEventWrite` trong `ntdll.dll` → tương tự
- Không ghi disk. Hiệu lực suốt vòng đời process implant.
- Được gọi tự động bởi `execute-shellcode` và `execute-assembly` trước khi thực thi.

**Source:** `implant/sliver/taskrunner/task_windows.go:patchAmsi`, `patchEtw`

---

### E2 — RefreshPE (EDR Hook Removal)

Xóa EDR inline hooks khỏi DLL đang load bằng cách ghi đè `.text` section từ bản sạch trên disk.

```
pe.Open(dll) → đọc .text từ disk
LoadDLL(dll) → lấy base address bản đang chạy
VirtualProtect(RWX) → mở quyền ghi lên .text in-memory
byte-by-byte copy → ghi đè hooks
VirtualProtect(restore) → khôi phục permissions
```

Áp dụng cho bất kỳ DLL nào (thường: `ntdll.dll`, `kernel32.dll`). Không có process mới, không có file write.

**Source:** `implant/sliver/evasion/evasion_windows.go:RefreshPE`

---

### E3 — PPID Spoofing

Giả mạo parent process của child process để ẩn trong process tree.

```
OpenProcess(PROCESS_CREATE_PROCESS, ppid)
→ SysProcAttr.ParentProcess = handle
→ CreateProcess → child xuất hiện dưới ppid trong tree
```

Dùng qua flag `--ppid <pid>` trong `execute`. Phát hiện: child process có parent không phải là creator thực sự.

**Source:** `implant/sliver/spoof/spoof_windows.go:SpoofParent`

---

### E4 — Build-Time Obfuscation

Áp dụng khi generate implant, trước khi deploy.

| Technique | Cơ chế | Flag |
|---|---|---|
| Symbol obfuscation | Garble: strip DWARF/build ID, randomize function/type names | `--obfuscate` |
| PE metadata clone | Copy `VersionInfo`, `IconGroup`, export table từ donor binary hợp lệ | `--spoof-metadata` |

Garble tự thêm `-s -w -buildid=` vào ldflags. Donor binary được chỉ định qua `SpoofMetadataConfig.PE.Source`.

**Source:** `server/generate/binaries.go`, `server/generate/spoof.go`

---

### E5 — Shellcode Encoding

Encode shellcode trước khi gửi/stage, áp dụng server-side.

| Encoder | Cơ chế |
|---|---|
| `XorEncoder` | XOR với static key, amd64/arm64 |
| `XorDynamicEncoder` | XOR với dynamic key tránh bad chars |
| `ShikataGaNai` | Polymorphic XOR (Shikata Ga Nai); self-decoding stub; signature thay đổi mỗi lần encode |

**Source:** `server/encoders/shellcode/shellcode.go`, `server/encoders/shellcode/sgn/sgn.go`

---

### E6 — Traffic Encoding (Runtime C2)

Custom transform trên C2 traffic trước khi gửi qua transport.

- WASM encoder module được load vào implant khi khởi động (build flag `TrafficEncodersEnabled`)
- Encoder pipeline áp dụng custom byte transforms lên mọi C2 data
- Chạy on top of mTLS / HTTP encryption — thêm một lớp obfuscation cho traffic pattern
- Operator thêm/xóa encoder qua `traffic-encoders` command

**Source:** `implant/sliver/encoders/traffic/traffic-encoder.go`, `implant/sliver/encoders/encoders.go`

---

### E7 — Beacon Jitter (Timing Evasion)

Áp dụng cho beacon mode (async implant).

```
sleep_duration = Interval + rand(0, Jitter)
```

Phá vỡ fixed-period beacon detection. Cấu hình khi generate:
```
generate beacon --sleep 60s --jitter 30s
```

**Source:** `implant/sliver/transports/beacon.go:Duration()`

---

### E8 — Execution Limits / Kill-Switch

Baked vào implant lúc compile. Implant tự exit nếu điều kiện không thỏa.

| Limit | Check | Config key |
|---|---|---|
| Anti-debug | `IsDebuggerPresent` → exit nếu ≠ 0 | luôn bật (non-debug build) |
| Hostname | `os.Hostname()` compare | `LimitHostname` |
| Username | `user.Current()` compare | `LimitUsername` |
| Domain-joined | `NetGetJoinInformation` | `LimitDomainJoined` |
| Datetime expiry | RFC3339 compare với `time.Now()` | `LimitDatetime` |
| File marker | `os.Stat(path)` → exit nếu không tồn tại | `LimitFileExists` |
| Locale | regex match vs system locale | `LimitLocale` |

**Source:** `implant/sliver/limits/limits.go:ExecLimits`, `limits_windows.go:PlatformLimits`

---

## Tổng quan nhanh

| Method | Spawn process mới | Disk write | Windows only | Beacon support |
|---|---|---|---|---|
| `execute` | Yes | No | No (--ppid/--token: Yes) | Yes |
| `execute-shellcode` | No (self) / Yes (remote) | No | Yes | Yes |
| `execute-assembly` | No | No | Yes | Yes |
| `sideload` / `spawndll` | Yes (sacrifice) | No | Yes | Yes |
| `execute-bof` | No | No | Yes | Yes |
| `wasm` | No | No | No | Yes |
| `shell` | Yes | No | No | **No** |
| `migrate` | No (inject) | No | Yes | No |

---

## Execution Path Flows

### Path A: Tiered shellcode execution (session)
```
execute-shellcode (self-inject, AMSI/ETW patched)
  ↓ T1620 + T1562.001
execute-assembly (CLR in-proc)
  ↓ T1620
execute-bof (BOF in-proc)
  ↓ T1620
```

### Path B: Lateral / privilege (Windows)
```
impersonate / make-token
  ↓ T1134.001
execute --token --ppid <explorer-pid>
  ↓ T1106 + T1134.004
migrate --pid <privileged-process>
  ↓ T1055.001
```

### Path C: Interactive recon
```
shell (PowerShell, piped, hidden)
  ↓ T1059.001
[Ctrl-] detach, chạy ngầm]
execute -o cmd.exe /c <discovery>
  ↓ T1106
```
