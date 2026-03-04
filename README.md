# Debugging AMD GPU Acceleration in NixOS-WSL

## TL;DR

**Problem:** `vulkaninfo` fails to detect the hardware-backed WSLg GPU, falling back to the slow `llvmpipe` software renderer.

**Root Cause:** A hidden dependency chain failure.
1. Mesa's D3D12 backend (`dozen`) fails to load `libd3d12.so`.
2. Even after providing the `libd3d12.so` library, it can detect the GPU but still fails to initialize it because the GPU driver (`amdxc64.so` in this case) provided by WSLg requires the `libssl.so` library to be loaded, but it is also not found in the library search path.

**Solution:**
Explicitly add both the WSL driver path and the OpenSSL path to `LD_LIBRARY_PATH`.

_Note: The OpenSSL library path may differ in your setup, so make sure to replace it with the correct path obtained from Nix._

```bash
LD_LIBRARY_PATH=/run/opengl-driver/lib:/nix/store/2p91cylbmdv4si5j818pnsg6qcbgin72-openssl-3.6.0/lib vulkaninfo --summary
```

---

## ⚠️ Environment & Setup

The Mesa version in this report is **`25.3.5`**.

This is the latest version at the time of writing.
The behavior may differ in other versions, so please check the corresponding source code if you are using a different version.
All the links in this report are also based on this version, so if you are using a different version, please check the corresponding source code for that version.

Also, here's my setup for reference:
- **NixOS (`nixos-version`):** 25.11.20260117.72ac591 (Xantusia)
- **GPU:** AMD Radeon RX 6700 XT
- **WSL (`wsl.exe --version`):**

  ```
  WSL version: 2.6.3.0
  Kernel version: 6.6.87.2-1
  WSLg version: 1.0.71
  MSRDC version: 1.2.6353
  Direct3D version: 1.611.1-81528511
  DXCore version: 10.0.26100.1-240331-1435.ge-release
  Windows version: 10.0.26200.7840
  ```

---

## Investigation

### Phase 1: Silent Fallback

Let's start with the most basic test: running `vulkaninfo --summary` to see if it can detect the GPU provided by WSLg and get its information correctly.

```bash
# Get vulkaninfo from nixpkgs
nix shell nixpkgs#vulkan-tools

# Run vulkaninfo
vulkaninfo --summary
```

**Expected Output:**

The output should show that there are two GPUs available:
1. `GPU0` is the actual GPU provided by WSLg (e.g., AMD Radeon RX 6700 XT) using the `Dozen` driver (Mesa's D3D12 backend).
2. `GPU1` is the software renderer `llvmpipe`.

**Actual Output:**

```
WARNING: [Loader Message] Code 0 : terminator_CreateInstance: Received return code -9 from call to vkCreateInstance in ICD /nix/store/wy8xvqjn24y2hhwp5j483hjdbb869nkf-mesa-25.3.5/lib/libvulkan_dzn.so. Skipping this driver.
...
Devices:
========
GPU0:
        deviceName         = llvmpipe (LLVM 21.1.8, 256 bits)
        driverName         = llvmpipe
        ...
```
The system completely skipped the hardware GPU and fell back to `llvmpipe` (software rendering).

### Phase 2: Tracing the D3D12 Failure

To understand why the driver was skipped, I used `strace` to intercept file system calls.

```bash
strace -v -e trace=%file,write -o vulkaninfo-summary-1.log vulkaninfo --summary
```

In the log, I found `libvulkan_dzn.so` searching for `libd3d12.so` and failing:

```
openat(AT_FDCWD, "/.../libd3d12.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
write(2, "terminator_CreateInstance: Recei"..., 187) = 187
```

So the flow of the error is like this:
- `libvulkan_dzn.so` (`dzn_device.c`):
  - Try loading `libd3d12.so`: Not found
    - `instance->d3d12_mod = util_dl_open(UTIL_DL_PREFIX "d3d12" UTIL_DL_EXT);`
  - Return code: `-9` (`VK_ERROR_INCOMPATIBLE_DRIVER`)

**Solution (Part 1):**
Provide the required library (`libd3d12.so`) in the search path. WSLg provides this in `/usr/lib/wsl/lib`. In NixOS-WSL with `wsl.useWindowsDriver = true`, these are also symlinked to `/run/opengl-driver/lib`.

```bash
LD_LIBRARY_PATH=/usr/lib/wsl/lib vulkaninfo --summary
# or
# LD_LIBRARY_PATH=/run/opengl-driver/lib vulkaninfo --summary
```

### Phase 3: AMD Driver Crash

After adding the library path, `vulkaninfo` failed with a different error:

```
MESA: error: ID3D12DeviceFactory::CreateDevice failed
WARNING: [../src/microsoft/vulkan/dzn_device.c:1137] Code 0 : VK_ERROR_INITIALIZATION_FAILED
ERROR: [Loader Message] Code 0 : setup_loader_term_phys_devs: Call to 'vkEnumeratePhysicalDevices' in ICD /nix/store/wy8xvqjn24y2hhwp5j483hjdbb869nkf-mesa-25.3.5/lib/libvulkan_dzn.so failed with error code -3
```

I ran `strace` again with the new environment variable.

```bash
LD_LIBRARY_PATH=/usr/lib/wsl/lib strace -v -e trace=%file,write -o vulkaninfo-summary-2.log vulkaninfo --summary
```

This time, `libd3d12.so` loaded fine. It also successfully loaded `libd3d12core.so`, `libdxcore.so` and `/dev/dxg`. However, deeper in the log:

```
openat(AT_FDCWD, "/usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so", O_RDONLY|O_CLOEXEC) = 7
...
openat(AT_FDCWD, "/usr/lib/wsl/lib/libssl.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/.../libssl.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
write(2, "MESA: error: ID3D12DeviceFactory"..., 54) = 54
```

The Windows/WSL AMD driver (`amdxc64.so`) is loaded by `libd3d12core.so`. This proprietary driver has a runtime dependency on OpenSSL (`libssl.so`). Because this driver is provided by WSLg and expected to run on a normal FHS environment, it doesn't know about Nix stores and fails to find `libssl.so`. 

Confirming with a detailed trace:

```bash
LD_LIBRARY_PATH=/usr/lib/wsl/lib strace -f -tt -s 256 -yy -kk -e trace=openat,openat2 -o vulkaninfo-summary-2-update.log vulkaninfo --summary
```
The stack trace showed `amdxc64.so` calling `dlopen` for `libssl.so`.

### Phase 4: Final Resolution

To solve this, provide the OpenSSL library path in `LD_LIBRARY_PATH`.

Get the path using:

```bash
nix eval --raw --expr 'with (import (builtins.getFlake "nixpkgs") {}); lib.makeLibraryPath [ openssl ]' --impure
# Or
nix-instantiate --raw --eval -E 'with (import <nixpkgs> {}); lib.makeLibraryPath [ openssl ]'
```

Combining everything:

```bash
LD_LIBRARY_PATH=/usr/lib/wsl/lib:/nix/store/2p91cylbmdv4si5j818pnsg6qcbgin72-openssl-3.6.0/lib vulkaninfo --summary
```

---

## Appendix A: Architecture Deep Dive

### The Dependency Chain

The hardware acceleration in WSLg relies on a specific chain of hand-offs:

1.  **Application:** `vulkaninfo`
2.  **Mesa Loader:** `libvulkan_dzn.so` (The "Dozen" Vulkan driver)
    *   *Requirement:* Needs `libd3d12.so` (forwarding to `libd3d12core.so`).
3.  **DX12 Loader/Core:** `libd3d12.so` & `libd3d12core.so`
    *   `libdxcore.so`: Abstraction layer between Windows and Linux kernel communication (IOCTLs).
    *   `libd3d12core.so`: Actual implementation of DX12 API.
    *   `libd3d12.so`: Thin loader forwarding calls to `libd3d12core.so`.
4.  **Vendor Driver:** `amdxc64.so` (In `/usr/lib/wsl/drivers/...`)
    *   *Action:* Loaded by `libd3d12core.so`.
    *   *Requirement:* **Needs `libssl.so`**.

### Why `nix-ld` Didn't Work

`nix-ld` works with ***unpatched*** binaries by providing a dynamic linker at standard paths. However, it doesn't work with **patched** binaries (like `vulkaninfo` from Nixpkgs) because they have already been patched with `patchelf`/`autoPatchelfHook` to have a hardcoded `RPATH` (not `RUNPATH`!). The dynamic linker prefers `RPATH` over `LD_LIBRARY_PATH` (unless configured otherwise), and `nix-ld` targets binaries that lack an interpreter/RPATH entirely.

## Appendix B: Development Shell (`mkShell`) Example

Note: Remember to remove any `/lib` suffix from the string paths, it will be added automatically by `lib.makeLibraryPath`.

```nix
mkShell {
  LD_LIBRARY_PATH = lib.makeLibraryPath [
    "/usr/lib/wsl" # or "/run/opengl-driver"
    pkgs.openssl
  ];
};
```

---

## References

- [Microsoft Blog: DirectX Heart Linux](https://devblogs.microsoft.com/directx/directx-heart-linux/)
- [Microsoft Blog: Getting Started with DX12 Agility SDK](https://devblogs.microsoft.com/directx/gettingstarted-dx12agility/)
- [WSLg Architecture - Hardware Accelerated OpenGL](https://devblogs.microsoft.com/commandline/wslg-architecture/#hardware-accelerated-opengl)
- [Mesa GitLab: D3D12 Agility SDK support](https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18305)
- [GPU selection in WSLg](https://github.com/microsoft/wslg/wiki/GPU-selection-in-WSLg)

---

*This report was structured, grammar-checked, and refined with the assistance of an LLM to ensure clarity and professional presentation.*

---
