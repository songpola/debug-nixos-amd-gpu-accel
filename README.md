# Debugging Flow

## TL;DR

### Problem

`vulkaninfo` cannot detect the GPU provided by WSLg and falls back to the software renderer (`llvmpipe`), which is very slow and not what we want.

### Cause

1. The `d3d12` backend of Mesa (which is used to access the GPU provided by WSLg) requires the `libd3d12.so` library to be loaded, but it is not found in the library search path, so it fails to initialize and falls back to the software renderer.
2. After providing the `libd3d12.so` library, it can detect the GPU but still fails to initialize it because the GPU driver (`amdxc64.so` in this case) provided by WSLg requires the `libssl.so` library to be loaded, but it is also not found in the library search path.

### Solution

1. Provide the required `libd3d12.so` library in the library search path by setting the `LD_LIBRARY_PATH` environment variable to include the path where `libd3d12.so` is located (e.g., `/run/opengl-driver/lib` or `/usr/lib/wsl/lib`).
2. Provide the required `libssl.so` library in the library search path by setting the `LD_LIBRARY_PATH` environment variable to include the path where `libssl.so` is located (e.g., the OpenSSL library path obtained from Nix).

In the end, a one-liner command to run `vulkaninfo` with the correct library search path would look like this:

_Note: The OpenSSL library path may differ in your setup, so make sure to replace it with the correct path obtained from Nix._

```
LD_LIBRARY_PATH=/run/opengl-driver/lib:/nix/store/2p91cylbmdv4si5j818pnsg6qcbgin72-openssl-3.6.0/lib vulkaninfo --summary
```

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

## Overview

Let's start with the most basic test: running `vulkaninfo --summary` to see if it can detect the GPU provided by WSLg and get its information correctly.

```bash
# Get vulkaninfo from nixpkgs
nix shell nixpkgs#vulkan-tools

# Run vulkaninfo
vulkaninfo --summary
```

### Expected Output

```
...
Devices:
========
GPU0:
        apiVersion         = 1.2.328
        driverVersion      = 25.3.5
        vendorID           = 0x1002
        deviceID           = 0x73df
        deviceType         = PHYSICAL_DEVICE_TYPE_DISCRETE_GPU
        deviceName         = Microsoft Direct3D12 (AMD Radeon RX 6700 XT)
        driverID           = DRIVER_ID_MESA_DOZEN
        driverName         = Dozen
        driverInfo         = Mesa 25.3.5
        conformanceVersion = 0.0.0.0
        deviceUUID         = ecc3ff5e-472a-32ab-50a6-bb313fecfa14
        driverUUID         = 150e2caf-05c7-660e-c1be-680cc003b5a4
GPU1:
        apiVersion         = 1.4.328
        driverVersion      = 25.3.5
        vendorID           = 0x10005
        deviceID           = 0x0000
        deviceType         = PHYSICAL_DEVICE_TYPE_CPU
        deviceName         = llvmpipe (LLVM 21.1.8, 256 bits)
        driverID           = DRIVER_ID_MESA_LLVMPIPE
        driverName         = llvmpipe
        driverInfo         = Mesa 25.3.5 (LLVM 21.1.8)
        conformanceVersion = 1.3.1.1
        deviceUUID         = 6d657361-3235-2e33-2e35-000000000000
        driverUUID         = 6c6c766d-7069-7065-5555-494400000000
```

The expected output should show that there are two GPUs available:
1. `GPU0` is the actual GPU provided by WSLg, which is my AMD Radeon RX 6700 XT in this case, and it is using the `Dozen` driver (which is the D3D12 backend of Mesa).
2. `GPU1` is the software renderer provided by Mesa, which is using the `llvmpipe` driver.

_This is also the expected final output that we want to achieve at the end of this debugging process._

### Actual Output

```
WARNING: [Loader Message] Code 0 : terminator_CreateInstance: Received return code -9 from call to vkCreateInstance in ICD /nix/store/wy8xvqjn24y2hhwp5j483hjdbb869nkf-mesa-25.3.5/lib/libvulkan_dzn.so. Skipping this driver.
...
Devices:
========
GPU0:
        apiVersion         = 1.4.328
        driverVersion      = 25.3.5
        vendorID           = 0x10005
        deviceID           = 0x0000
        deviceType         = PHYSICAL_DEVICE_TYPE_CPU
        deviceName         = llvmpipe (LLVM 21.1.8, 256 bits)
        driverID           = DRIVER_ID_MESA_LLVMPIPE
        driverName         = llvmpipe
        driverInfo         = Mesa 25.3.5 (LLVM 21.1.8)
        conformanceVersion = 1.3.1.1
        deviceUUID         = 6d657361-3235-2e33-2e35-000000000000
        driverUUID         = 6c6c766d-7069-7065-5555-494400000000
```

## Issue 1: `... Received return code -9 from call to vkCreateInstance ...`

```
WARNING: [Loader Message] Code 0 : terminator_CreateInstance: Received return code -9 from call to vkCreateInstance in ICD /nix/store/wy8xvqjn24y2hhwp5j483hjdbb869nkf-mesa-25.3.5/lib/libvulkan_dzn.so. Skipping this driver.
...
```

### Cause

We can use `strace` to trace the system calls made by `vulkaninfo` to a file like this:

> Personally, I found myself a better alternative: [`lurk`](https://github.com/JakWai01/lurk) \
> But for the sake of simplicity, let's just use `strace` here.
>
> Note: If you use `lurk`, the format of the output may be different from `strace`, so adjust the following commands accordingly.

```
strace -v -e trace=%file,write -o vulkaninfo-summary-1.log vulkaninfo --summary
```

This will run the `vulkaninfo --summary` command and trace all the _file-related_ system calls (like `open`, `read`, `write`, etc.)
and the _`write`_ system call (which are used to print the error messages) to the `vulkaninfo-summary-1.log` file.
(You can name the file whatever you like.)

In the file, you will find a lot of lines like this:

```
...
openat(AT_FDCWD, "/.../libd3d12.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
```

This indicates that `vulkaninfo` (or more specifically, `libvulkan_dzn.so`) is trying to load `libd3d12.so` but fails to find it in the library search path.

Following with the `write` system calls, which print the log messages we saw earlier:

```
...
write(2, "terminator_CreateInstance: Recei"..., 187) = 187
...
```

So the flow of the error is like this:

`libvulkan_dzn.so` (`dzn_device.c`):
- Try loading `libd3d12.so`: Not found
  - `instance->d3d12_mod = util_dl_open(UTIL_DL_PREFIX "d3d12" UTIL_DL_EXT);`
    - https://gitlab.freedesktop.org/mesa/mesa/-/blob/25.3/src/microsoft/vulkan/dzn_device.c?ref_type=heads#L1836
- Return code: `-9` (`VK_ERROR_INCOMPATIBLE_DRIVER`)
  - `return vk_error(NULL, VK_ERROR_INCOMPATIBLE_DRIVER);`
    - https://gitlab.freedesktop.org/mesa/mesa/-/blob/25.3/src/microsoft/vulkan/dzn_device.c?ref_type=heads#L1839
  - `VK_ERROR_INCOMPATIBLE_DRIVER = -9,`
    - https://gitlab.freedesktop.org/mesa/mesa/-/blob/25.3/include/vulkan/vulkan_core.h#L155

### Solution

Provide the required library (`libd3d12.so`) in the library search path, which can be done by setting the `LD_LIBRARY_PATH` environment variable to include the path where `libd3d12.so` is located.

WSLg provides `libd3d12.so` in the `/usr/lib/wsl/lib` directory, so we can set the `LD_LIBRARY_PATH` environment variable to include this path:

```
LD_LIBRARY_PATH=/usr/lib/wsl/lib vulkaninfo --summary
```

Also, in NixOS-WSL, `libd3d12.so` is provided by the [`wsl.useWindowsDriver`](https://nix-community.github.io/NixOS-WSL/options.html#wslusewindowsdriver) option.
This will symlink the contents of `/usr/lib/wsl/lib` to `/run/opengl-driver/lib` (by adding a package containing the symlinks to `hardware.graphics.extraPackages`).

So if you're using NixOS-WSL and have enabled the `wsl.useWindowsDriver` option, you can set the `LD_LIBRARY_PATH` environment variable to include `/run/opengl-driver/lib` instead:

```
LD_LIBRARY_PATH=/run/opengl-driver/lib vulkaninfo --summary
```

## Issue 2: `ID3D12DeviceFactory::CreateDevice failed` and `VK_ERROR_INITIALIZATION_FAILED`

```
...
WARNING: dzn is not a conformant Vulkan implementation, testing use only.
Dropped Escape call with ulEscapeCode : 0x03007703
MESA: error: ID3D12DeviceFactory::CreateDevice failed
WARNING: [../src/microsoft/vulkan/dzn_device.c:1137] Code 0 : VK_ERROR_INITIALIZATION_FAILED
ERROR: [Loader Message] Code 0 : setup_loader_term_phys_devs: Call to 'vkEnumeratePhysicalDevices' in ICD /nix/store/wy8xvqjn24y2hhwp5j483hjdbb869nkf-mesa-25.3.5/lib/libvulkan_dzn.so failed with error code -3
...
```

### Cause

Same as before, let's use `strace` to trace the system calls,
this time with the `LD_LIBRARY_PATH` environment variable set,
and output to a different log file to avoid confusion:

```
LD_LIBRARY_PATH=/run/opengl-driver/lib strace -v -e trace=%file,write -o vulkaninfo-summary-2.log vulkaninfo --summary
```

Note that this time `libd3d12.so` is found and loaded successfully:

```
...
openat(AT_FDCWD, "/run/opengl-driver/lib/libd3d12.so", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/run/opengl-driver/lib/libd3d12core.so", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/run/opengl-driver/lib/libdxcore.so", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/dev/dxg", O_RDONLY|O_CLOEXEC) = 3
...
```

Additionally, it also load `libd3d12core.so`, `libdxcore.so` and the `/dev/dxg` device successfully, which are required for GPU acceleration in WSLg. You can learn more about these libraries and the device in the [Microsoft's blog post](https://devblogs.microsoft.com/directx/directx-heart-linux/#dxcore-&-d3d12-on-linux).

Now, back to our problem, the error message is found here:

```
...
write(2, "WARNING: dzn is not a conformant"..., 74) = 74
openat(AT_FDCWD, "/usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so", O_RDONLY|O_CLOEXEC) = 7
...
openat(AT_FDCWD, "/run/opengl-driver/lib/libssl.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/.../libssl.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
...
write(2, "MESA: error: ID3D12DeviceFactory"..., 54) = 54
...
write(2, "VK_ERROR_INITIALIZATION_FAILED", 30) = 30
...
```

From the log, we can see that:
1. `libvulkan_dzn.so` successfully loaded the `amdxc64.so`, which is the actual GPU driver provided by my GPU vendor (AMD in this case).
   - The driver is located in the `/usr/lib/wsl/drivers/` directory, which is where WSLg mounts the Windows GPU drivers.
   - ~~I still don't know how `libvulkan_dzn.so` finds the driver, but I _guess_ that one of the libraries (`libd3d12.so`, `libd3d12core.so` or `libdxcore.so`) is responsible for loading the driver, as they are the ones provided by WSLg and I don't see any reference to the drivers directory in the source code of Mesa. (Again, see the [Microsoft's blog post](https://devblogs.microsoft.com/directx/directx-heart-linux/#dxcore-&-d3d12-on-linux) for its architecture.)~~
   - **UPDATE:** After further investigation, I found that it is actually `libd3d12core.so` that loads the driver (`amdxc64.so`). \
     You can check with this command:
     
     ```
     LD_LIBRARY_PATH=/run/opengl-driver/lib strace -f -tt -s 256 -yy -kk -e trace=openat,openat2 -o vulkaninfo-summary-2-update.log vulkaninfo --summary
     ```
     Output (abbreviated):
     ```
     ...
     ... openat(..., "/usr/lib/wsl/drivers/.../amdxc64.so", O_RDONLY|O_CLOEXEC) = 5</usr/lib/wsl/drivers/.../amdxc64.so>
      ...
      > /nix/store/wb6rhpznjfczwlwx23zmdrrw74bayxw4-glibc-2.42-47/lib/libc.so.6(dlopen@GLIBC_2.2.5+0x6a) [0x99bea]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x319b05]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x44291e]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x42e554]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x35db2b]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x35d87f]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x35e488]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x35fe3f]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x318962]
      > /nix/store/wy8xvqjn24y2hhwp5j483hjdbb869nkf-mesa-25.3.5/lib/libvulkan_dzn.so(d3d12_create_device+0x72) [0x70ca2]
     ...
     ```
2. ~~Maybe~~, the `amdxc64.so` was trying to load `libssl.so` but failed to find it in the library search path.
   - It also tries to find in the `/run/opengl-driver/lib` directory (as we added to `LD_LIBRARY_PATH`) first, but it is not found there either.
   - ~~I'm still not sure whether the library is actually required by the GPU driver (`amdxc64.so`) or the Mesa's  `libvulkan_dzn.so`, but I can't find any reference to `libssl.so` in the source code of `libvulkan_dzn.so`, so it is likely that it is required by the driver (`amdxc64.so`).~~
   - **UPDATE:** Confirmed that it is indeed required by the GPU driver (`amdxc64.so`), found from the same log as above:
     
     ```
     ...
     ... openat(..., "/run/opengl-driver/lib/libssl.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
      ...
      > /nix/store/wb6rhpznjfczwlwx23zmdrrw74bayxw4-glibc-2.42-47/lib/libc.so.6(dlopen@GLIBC_2.2.5+0x6a) [0x99bea]
      > /usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so() [0x3ed3da4]
      > /usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so() [0x3ed4174]
      > /usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so() [0x3ed4487]
      > /usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so() [0x3ed3be1]
      > /usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so() [0x3ede180]
      > /usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so() [0x9c7ae3]
      > /usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so() [0x9c87a0]
      > /usr/lib/wsl/drivers/u0198633.inf_amd64_f6ca42135bb94c90/B025777/amdxc64.so() [0x9c8f17]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x442780]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x42e6a7]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x35db2b]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x35d87f]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x35e488]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x35fe3f]
      > /usr/lib/wsl/lib/libd3d12core.so() [0x318962]
      > /nix/store/wy8xvqjn24y2hhwp5j483hjdbb869nkf-mesa-25.3.5/lib/libvulkan_dzn.so(d3d12_create_device+0x72) [0x70ca2]
     ...
     ```
3. Finally, it gave up and returned the `ID3D12DeviceFactory::CreateDevice failed` error, which is then translated to `VK_ERROR_INITIALIZATION_FAILED` (`-3`).

### Solution

This problem can be solved simply by providing the required OpenSSL library (`libssl.so`) in the library search path.

To get the path of the OpenSSL library, you can use one of the following commands:
- If you have enabled the `nix-command` and `flakes` experimental features in your Nix/NixOS:
  ```bash
  nix eval --raw --expr 'with (import (builtins.getFlake "nixpkgs") {}); lib.makeLibraryPath [ openssl ]' --impure
  ```
- Or, with `nix-instantiate` command:
  ```bash
  nix-instantiate --raw --eval -E 'with (import <nixpkgs> {}); lib.makeLibraryPath [ openssl ]'
  ```

The result should be something like this:

```
/nix/store/2p91cylbmdv4si5j818pnsg6qcbgin72-openssl-3.6.0/lib
```

With the path, we can set the `LD_LIBRARY_PATH` environment variable to include it:

```
LD_LIBRARY_PATH=/run/opengl-driver/lib:/nix/store/2p91cylbmdv4si5j818pnsg6qcbgin72-openssl-3.6.0/lib vulkaninfo --summary
```

## Summary

In this report, we have debugged the issue of `vulkaninfo` not being able to detect the GPU provided by WSLg and falling back to the software renderer. We found that the issue was caused by missing libraries (`libd3d12.so` and `libssl.so`) in the library search path, which are required by the D3D12 backend of Mesa and the GPU driver provided by WSLg, respectively. By providing these libraries in the library search path, we were able to successfully detect and initialize the GPU provided by WSLg.

### Example `mkShell` configurations

Note: Remember to remove any `/lib` suffix from the string paths, it will be added automatically by `lib.makeLibraryPath`.

```nix
mkShell {
  LD_LIBRARY_PATH = lib.makeLibraryPath [
    "/usr/lib/wsl" # or "/run/opengl-driver"
    pkgs.openssl
  ];
};
```

## Extra: How about using `nix-ld`?

Short answer: Yes, and, No.

It works with ***unpatched*** binaries, but it doesn't work with **patched** binaries (like `vulkaninfo` from Nixpkgs) because they have already been patched with `patchelf`/`autoPatchelfHook` to have `RPATH` (not `RUNPATH`) pointed to the required libraries, so they will not look for the libraries in the `LD_LIBRARY_PATH` environment variable.

`nix-ld` is a tool that allows you to run ***unpatched*** dynamically linked executables with libraries from a Nix store path. It works by creating a wrapper script of a dynamic linker (specified by `NIX_LD` environment variable) that sets the `LD_LIBRARY_PATH` environment variable to the value of `NIX_LD_LIBRARY_PATH` environment variable. This allows the executable to find the required libraries at runtime without needing to modify the system-wide configuration or the binary itself.

---

**My Personal References**

- https://renenyffenegger.ch/notes/Windows/Subsystem-for-Linux/index
- https://github.com/microsoft/WSL2-Linux-Kernel/blob/linux-msft-wsl-6.6.y/drivers/hv/dxgkrnl/Kconfig
  - ```
      When such container is instantiated, the Windows host
      assigns compatible host GPU adapters to the container. The corresponding
      virtual GPU devices appear on the PCI bus in the container. These
      devices are enumerated and accessed by this driver.

      Communications with the driver are done by using the Microsoft libdxcore
      library, which translates the D3DKMT interface
      <https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/d3dkmthk/>
      to the driver IOCTLs. The virtual GPU devices are paravirtualized,
      which means that access to the hardware is done in the host. The driver
      communicates with the host using Hyper-V VM bus communication channels.
    ```
- https://devblogs.microsoft.com/directx/directx-heart-linux/
- https://devblogs.microsoft.com/commandline/wslg-architecture/#hardware-accelerated-opengl
- https://github.com/microsoft/wslg/wiki/GPU-selection-in-WSLg
  - > Two key take aways of this article for me were:
    > - Newer versions of MESA with the d3d12 backend choose the first enumerated integrated GPU.
    > - The environment variable `MESA_D3D12_DEFAULT_ADAPTER_NAME` can be set to a substring of the name of the GPU that should be selected
    > - `glxinfo -B` shows which GPU is currently used
- https://learn.microsoft.com/en-us/archive/blogs/wsl/pico-process-overview

- https://renenyffenegger.ch/notes/Windows/Subsystem-for-Linux/index
- https://wsldl-pg.github.io/ArchW-docs/Known-issues/#intel-graphics
  - https://github.com/yuk7/ArchWSL/issues/308

- ```
  - The openat on /usr/lib/wsl/drivers/.../amdxc64.so is issued during a dlopen path inside glibc loader, and the first non-loader caller is libd3d12core.so.
  - In Mesa, the triggering callsite is d3d12_create_device from dzn_util.c, reached from dzn_device.c.
  - Your stack also shows that exact chain: d3d12_create_device → dzn_physical_device_create → dzn_enumerate_physical_devices_dxcore → vkEnumeratePhysicalDevices.
  - So the actual choice of /usr/lib/wsl/drivers/.../amdxc64.so is made inside libd3d12core.so, not by Mesa dzn code.
  - This happens while physical devices are being enumerated (vulkaninfo calling vkEnumeratePhysicalDevices), not later during queue/device operation.
  ```

- **How GPU acceleration works in WSLg**

  The WSLg GPU driver, auto-mounted under `/usr/lib/wsl/lib`, consists of:

  - `libdxcore.so`: Its main job is to **abstract** the differences between how Windows and Linux communicate with the kernel. While Windows uses a service table, Linux uses IOCTLs; `libdxcore.so` handles these differences so the API and drivers can remain consistent across platforms. See [^1] for more details.
  - `libd3d12core.so`: Introduced in DirectX 12 Agility SDK. This is the *actual* implementation of DirectX 12 API. See [^2] and [^3] for more details.
  - `libd3d12.so`: The main DirectX 12 library. Since the introduction of DirectX 12 Agility SDK, this library is now just a thin loader that forwards calls to the actual implementation in `libd3d12core.so` (see [^2] for more details).

  [^1]: https://devblogs.microsoft.com/directx/directx-heart-linux/#dxcore-&-d3d12-on-linux

  [^2]: https://devblogs.microsoft.com/directx/gettingstarted-dx12agility/

  [^3]: https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/18305

  The application (or API/library) that wants utilize GPU acceleration will call into `libd3d12.so`, which forwards the calls to `libd3d12core.so`. The `libd3d12core.so` then calls into `libdxcore.so` to communicate with the kernel and perform the necessary operations.

- **Resolving required library dependencies**

  The application also needs to know where to find the required libraries (loaded via `dlopen` function) at runtime. There are two common ways to specify the library search path:

  - Compile time: Hardcoded path in the binary (`RPATH`/`RUNPATH`), which can be inspected using `ldd`.
  - Runtime: `LD_LIBRARY_PATH` environment variable, (which will be used by the dynamic linker).

  Also, sereral graphics-related libraries (OpenGL[^4], Vulkan[^5], etc.) are also required. In NixOS, these libraries are mounted under `/run/opengl-driver/lib` (and `/run/opengl-driver-32/lib`[^6]), so we also need to include this path in the `LD_LIBRARY_PATH` environment variable:

  ```
  LD_LIBRARY_PATH=/run/opengl-driver/lib
  # or
  LD_LIBRARY_PATH=/run/opengl-driver/lib:/run/opengl-driver-32/lib
  ```

  [^4]: https://wiki.nixos.org/wiki/OpenGL
  [^5]: https://wiki.nixos.org/wiki/Graphics#Vulkan
  [^6]: https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/hardware/graphics.nix

  This can be done in multiple ways, such as:

  - System-wide NixOS option: `environment.variables.LD_LIBRARY_PATH`
  - Per-user shell configuration: `~/.bashrc`, `~/.zshrc`, etc.
  - Per-application wrapper script: A custom script that sets the environment variable and then launches the application.
  - Per-instance environment variable: Setting the variable directly in the terminal before launching the application.
  - Using `patchelf` to modify the binary's `RPATH`/`RUNPATH` to include the required library paths.
  - **Or, use [`nix-ld`](https://github.com/nix-community/nix-ld)**, which will be discussed in the next section.

- **Digging into how `nix-ld` works**

  Let's look at its implementation in the [`nix-ld` NixOS modules](https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/programs/nix-ld.nix).
  (This is just a simplified version for brevity.)

  ```nix
  let
    # 1
    nix-ld-libraries = pkgs.buildEnv {
      name = "lb-library-path";
      pathsToLink = [ "/lib" ];
      paths = map lib.getLib cfg.libraries;
      # TODO make glibc here configurable?
      postBuild = ''
        ln -s ${pkgs.stdenv.cc.bintools.dynamicLinker} $out/share/nix-ld/lib/ld.so
      '';
      extraPrefix = "/share/nix-ld";
      ignoreCollisions = true;
    };
  in
  {
    # 2
    environment.ldso = "${cfg.package}/libexec/nix-ld";

    # 3
    environment.systemPackages = [ nix-ld-libraries ];
    environment.pathsToLink = [ "/share/nix-ld" ];

    # 4
    environment.variables = {
      NIX_LD = "/run/current-system/sw/share/nix-ld/lib/ld.so";
      NIX_LD_LIBRARY_PATH = "/run/current-system/sw/share/nix-ld/lib";
    };
  }
  ```

  1. Create a derivation, `nix-ld-libraries`, that builds an environment with the required libraries (specified in `programs.nix-ld.libraries`) and the dynamic linker (`ld.so`).
  2. Use `nix-ld` as the dynamic linker for the system. For instance, this will create a symlink at `/lib64/ld-linux-x86-64.so.2`. This is also the secret of how `nix-ld` allows you to run unpatched binaries in NixOS: Unpatched binaries will look for the dynamic linker at `/lib64/ld-linux-x86-64.so.2` (or similar path depending on the architecture), which is now a symlink to the `nix-ld` wrapper script. Without setting the `environment.ldso` option (NixOS default is `null`), unpatched binaries will not be able to find the dynamic linker and will fail to run.
  3. Add the derivation to the system closure and make the libraries available at `/run/current-system/sw/share/nix-ld`.
  4. Set the `NIX_LD` and `NIX_LD_LIBRARY_PATH` environment variables to point to the actual dynamic linker and library path, respectively.
