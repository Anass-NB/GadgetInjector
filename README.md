# GadgetInjector 
Patch an iOS `.ipa` to auto-load Frida Gadget. No jailbreak needed. For authorized testing on apps you own — that's it, that's the rule.


---

## What it does

```
MyApp.ipa  -->  unzip  -->  patch Mach-O (add rpath + load gadget)  -->  repack  -->  MyApp-frida-listen.ipa
```

1. Finds the `.app` + main binary inside `Payload/`
2. Resolves a Frida version (auto / `--frida-version` / `--prefer-latest`)
3. Downloads matching `FridaGadget.dylib`
4. Checks arm64/arm64e arch compat
5. Injects rpath + load command via LIEF
6. Drops `FridaGadget.config` (TCP listen mode) + `bootstrap.js`
7. Strips old `_CodeSignature/` + `SC_Info/`
8. Repacks IPA

**Does NOT**: sign, install, or launch. That's on you.

---

## Install

```powershell
uv venv
.venv\Scripts\activate
uv pip install lief frida frida-tools pymobiledevice3
```
---

## Usage

```bash
python3 Gadget_injector.py MyApp.ipa
```

| Flag | What |
|---|---|
| `-o, --output` | output path |
| `--frida-version` | pin a version (default: auto-detect host) |
| `--prefer-latest` | always grab newest Frida release |
| `--gadget-name` | rename the dylib (default `FridaGadget.dylib`) |
| `--port` | listen port (default `27042`) |
| `--bundle-id` | target app if IPA has multiple |
| `--force` | overwrite / re-inject |
| `--no-clean` | keep old codesign artifacts |
| `--vibe` | banner style: cyberpunk / hacker / block / stealth / glitch / minimal |
| `--no-banner` | shut it up (CI) |
| `--debug` | verbose: rpaths, libs, arch, version resolution |

---

## After patching (Windows)

1. **Resign + install** — pick one:
   - [iLoader](https://github.com/nab138/iloader) — open source, handles Apple ID sign-in + pairing + install
   - Sideloadly
   - AltStore
   - CLI: `zsign` or `rcodesign`
   - ❌ Xcode / `codesign` / `xcrun devicectl` — macOS only, won't work here

2. **Launch** the app normally (no `devicectl` paused-launch on Windows)

3. **Forward the port**:
   ```bash
   pymobiledevice3 usbmux forward 27042 27042
   ```

4. **Attach**:
   ```bash
   objection -N -h 127.0.0.1 -p 27042 explore
   # or
   frida -H 127.0.0.1:27042 -n <process-name>
   ```

---

## Known gotchas

- Free Apple ID signing → **7-day cert expiry**, max ~3 App IDs, no custom entitlements
- All embedded dylibs (incl. the Gadget) must share the **same Team ID**
- **Never** add entitlements to `FridaGadget.dylib` itself
- Arch mismatch → check with `--debug`, confirm gadget build has the slice your device needs (arm64e on modern hardware)
- Gadget not loading at runtime → likely signing/Team ID issue, not an injection issue