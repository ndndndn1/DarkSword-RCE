# DarkSword iOS Exploit Chain Analysis

Reference: https://cloud.google.com/blog/topics/threat-intelligence/darksword-ios-exploit-chain

> I'm not interested in the RU/UA politics, sloppy tradecraft results in exploits being found.

## Overview

DarkSword is a full-chain iOS exploit targeting iOS 18.4 through 18.6.2, covering iPhone XS (A12) to iPhone 16 Pro Max (A18 Pro). It chains four stages — WebKit RCE, sandbox escape via GPU process, privilege escalation, and data exfiltration — delivered through a single watering-hole page.

## File Structure

| File | Role | Lines |
|---|---|---|
| `index.html` | Entry point — creates hidden 1x1 iframe | 25 |
| `frame.html` | Loads `rce_loader.js` from C2 (`static.cdncounter.net`) | 9 |
| `rce_loader.js` | Orchestrator — detects iOS version, loads version-specific modules | 247 |
| `rce_module.js` | WebKit RCE module (iOS 18.4) — per-device offset table + memory R/W primitives | 3,485 |
| `rce_module_18.6.js` | iOS 18.6 stub (logic moved into worker) | 4 |
| `rce_worker.js` | Web Worker exploit engine (iOS 18.4) — IPC encoder, type confusion | 952 |
| `rce_worker_18.6.js` | Web Worker exploit engine (iOS 18.6) — self-contained with offsets | 10,205 |
| `sbx0_main_18.4.js` | Sandbox escape Stage 0 — WebKit → GPU process | 8,423 |
| `sbx1_main.js` | Sandbox escape Stage 1 — code execution in GPU process | 6,862 |
| `pe_main.js` | Privilege escalation — kernel-level attack + data exfiltration (webpack bundle) | 8,440 |

## Execution Chain

```
Victim visits watering-hole page
       │
       ▼
┌─────────────────────────────────────────────────┐
│  Stage 0: Entry Point (Main Thread)             │
│  index.html → frame.html → rce_loader.js        │
│                                                  │
│  • Hidden iframe injection                       │
│  • iOS version detection via User-Agent          │
│  • Dynamic loading of version-specific modules   │
│  • Web Worker + dlopen Worker setup              │
└──────────────────┬──────────────────────────────┘
                   │ postMessage('stage1_rce')
                   ▼
┌─────────────────────────────────────────────────┐
│  Stage 1: WebKit RCE (Web Worker)               │
│  rce_worker_18.6.js :: _aarw_main()             │
│                                                  │
│  A. JIT Warmup                                   │
│     _f2i/_i2f × 20K, get_oob × 300K             │
│     → DFG/FTL compilation trigger                │
│                                                  │
│  B. Type Confusion (UAF)                         │
│     treetab-based deep tree (depth=12)           │
│     → GC pressure → flatten → UAF               │
│     → contig/double array pair                   │
│     → addrof/fakeobj primitives                  │
│                                                  │
│  C. Arbitrary R/W                                │
│     fakeobj → scribble_element → write64         │
│     BigUint64Array + charCodeAt → read64         │
│                                                  │
│  D. Post-Exploitation Setup                      │
│     • Disable JSC GC (isSafeToCollect = 0)       │
│     • Mach-O header scan → jsc_base              │
│     • pthread linkedit → device fingerprinting   │
│     • ASLR slide calculation + offset resolution │
│     • JIT allowlist patch                        │
│     • dlopen/dlsym/signPointer PAC signing       │
│     • fcall primitive (arbitrary native calls)   │
└──────────────────┬──────────────────────────────┘
                   │ eval(sbx0_main_18.4.js)
                   ▼
┌─────────────────────────────────────────────────┐
│  Stage 2: Sandbox Escape (WebKit → GPU)         │
│  sbx0_main_18.4.js                              │
│                                                  │
│  • WebKit IPC message forgery (Encoder class)    │
│  • RemoteGraphicsContextGL manipulation          │
│  • GPU process memory R/W                        │
│  • gpuRead64/gpuWrite64/gpuPacia/gpuPacib        │
│  • gpuDlsym/gpuFcall → GPU function calls       │
└──────────────────┬──────────────────────────────┘
                   │ eval(sbx1_main.js)
                   ▼
┌─────────────────────────────────────────────────┐
│  Stage 3: GPU Process Exploitation              │
│  sbx1_main.js                                   │
│                                                  │
│  • func_resolve(dlsym) in GPU context            │
│  • Shared cache slide (syscall 294)              │
│  • JSGlobalContextCreate → new JSContext         │
│  • Download pe_main.js from C2                   │
│  • evaluateScript in GPU JSContext               │
└──────────────────┬──────────────────────────────┘
                   │ evaluateScript(pe_main.js)
                   ▼
┌─────────────────────────────────────────────────┐
│  Stage 4: Privilege Escalation + Exfiltration   │
│  pe_main.js (webpack bundle)                    │
│                                                  │
│  • launchd task port discovery                   │
│  • MIG filter bypass                             │
│  • sandbox_extension_issue_file                  │
│  • Jetsam memory limit removal                   │
│                                                  │
│  Target process injection (InjectJS):            │
│  ├─ SpringBoard    → loader.js                   │
│  ├─ securityd      → keychain_copier.js          │
│  ├─ wifid          → wifi_password_dump.js       │
│  ├─ securityd      → wifi_password_securityd.js  │
│  ├─ UserEventAgent → icloud_dumper.js            │
│  └─ SpringBoard    → file_downloader.js → C2     │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
          redirect → 404.html (cleanup)
```

## Process Boundary Transitions

| Transition | Technique | Primitive |
|---|---|---|
| JS → WebKit process | JSC type confusion (UAF) | addrof/fakeobj → read64/write64 |
| WebKit → GPU process | IPC message forgery (RemoteGraphicsContextGL) | gpuRead64/gpuWrite64/gpuFcall |
| GPU → launchd/kernel | Mach task port discovery + MIG filter bypass | RemoteCall + InjectJS |
| Each process → C2 | file_downloader.js | HTTP upload |

## Exfiltration Targets

- **Keychain** (via securityd) — stored passwords, certificates
- **WiFi passwords** (via wifid + securityd)
- **iCloud Drive files** (via UserEventAgent)
- **Forensic files** (via SpringBoard → file_downloader → C2)

## Target Coverage

- **iOS versions**: 18.4, 18.4.1, 18.5, 18.6, 18.6.1, 18.6.2
- **Devices**: iPhone XS (A12) through iPhone 16 Pro Max (A18 Pro), ~25 device models
- **Build IDs**: 22E240, 22E252, 22F76, 22G86, 22G90, 22G100
- **C2 server**: `static.cdncounter.net` (CDN-disguised)

## Key Techniques

- **PAC bypass**: `pacia`/`pacib` gadgets, `noPAC()` mask (`& 0x7ffffffffn`)
- **ASLR bypass**: dyld shared cache slide via `syscall 294`
- **Device fingerprinting**: `libsystem_pthread` linkedit address → device model lookup
- **ObjC class loading trick**: `ImageBitmap.close()` → `dlopen` via AVSpeechSynthesis/TextToSpeech
- **Symbol interposing**: CMPhoto/ImageIO/Security symbol table overwrite
- **Side-channel logging**: `fopen("/path/msg")` — encodes log in filesystem path

## Privacy & Security Notice

This repository contains analysis artifacts of a real-world exploit chain attributed to a state-sponsored threat actor. All code is provided strictly for **defensive security research and educational purposes**.

- No live C2 infrastructure is included or referenced
- Hardcoded offsets are version-specific and non-functional without matching firmware
- The exploit chain targets vulnerabilities that have been patched by Apple
- Do not deploy, weaponize, or use this code against systems without explicit authorization
