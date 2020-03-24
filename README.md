# KVM-Opencore
Opencore Configuration of KVM Hackintosh with tweaks

# Status

* OS: tested in Catalina 10.15.x
* Opencore: 0.5.6
    * NVRAM: native
    * AppleHotKey: works
    * FileVault: un-tested
    * Boot-Audio: works with onboard audio passthrough
    * OVMF: you can use latest version other than the old way(patched one). see issues.
* SMBIOS: iMacPro1,1
* NIC: vmxnet3/e1000-82545em/passthrough
* CPU:
    * Penryn: works, using patches to enable leaf7 support for better performance
    * Intel Host-Passthrough: works, but with patches and remove topology line
    * AMD Host-PassThrough: works, but with AMD-Vanilla patches
* GPU Passthrough: works
    * HDMI/DP Audio: works
    * Metal Support: works
    * H264 Hardware Decoding: works
    * H265 Hardware Decoding: works
* Handoff: works with specific Wifi/Bluetooth
* USB(Passthrough): works
* Power Manages
  * Sleep: works, see details below.
  * EC: faked one
  * USB Power: works even with USB3 passthrough
  * CPU AGPM: wo don't need CPU AGPM because hypervisor handled it well, this is for GPU AGPM
  * GPU AGPM: works

# Usage
1. Download the repository and put the EFI under your EFI disk
2. Download latest Lilu/WhatEverGreen/AppleALC and put them into kext folder, also add them to config.plist

# Tips
## SMBios
Comparing with normal hackintosh setup, we only have limited choices here(because we don't have iGPU)
* iMacPro1,1(Prefer)
* MacPro7,1
## CPU
You have two options to choose: Penryn, Passthrough
### Penryn
Penryn is the safest choice in Hackintosh VM, it is classic, and Apple wrote native code to support it.
But we should do something to make it work more than that:

1. You should passthrough some important cpuid features which don't included in Penryn Model, such as:
   * invtsc: latest MacOS won't boot without it
   * AVX: latest MacOS won't boot without it
   * FMA: metal support needs this
   * AVX2
   * BMI1
   * BMI2
   * kvm=on: QEMU use it to expose Hypervisor leaf node, which MacOS use it to determine the invtsc frequency from hypervisor node.
   * ~~vmware-cpu-freq=on~~: no more need, QEMU enabled it by default.
2. Add patch(from AMD-Vanilla) to support leaf7 cpuid features support
   * otherwise App won't use some important feature like AVX
   * can be check with `sysctl -a|grep machdep.cpu.leaf7_features` to see whether AVX/AVX2 is included.

__NOTES__: Topology/Hyper-Threading is working well with Penryn, but you can't not use odd num(like 3/5/7 and greater than 8) cores.

### Passthrough
We can't use topology when using cpu passthrough, detail/discussion can be found in issues.
So remove the line(`<topology sockets='1' cores='4' threads='2'/>`) from your libvirt xml.
#### Intel
If you are using most recent generation Intel CPU(newer than Penryn), then you can just just passthrough you cpu type with `mode='host-passthrough'`.

#### AMD
You can passthrough amd process too, but with AMD-Vanilla patches like bare metal hackintosh.

## GPU
GPU passthrough is a good way to gain smooth experience in MacOS.If you do so, make sure you doing things right.
__You have to make sure you put the gfx and graphic audio in the same bus but different function, otherwise DP/HDMI audio, Metal Support, HW Accuration won't work__.
```xml
<!-- Assume we have a graphic(gfx(0x2d/0x0) and audio(0x2d/0x1)) -->
<hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio'/>
    <source>
        <address domain='0x0000' bus='0x2d' slot='0x00' function='0x0'/>
    </source>
<!-- we put gfx under bus 0x01 and function 0x0, also with multifunction on. -->
    <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0' multifunction='on'/>
</hostdev>
<hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio'/>
    <source>
        <address domain='0x0000' bus='0x2d' slot='0x00' function='0x1'/>
    </source>
<!-- we put graphic audio under the same bus 0x01 with gfx but different function 0x1-->
    <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
</hostdev>
```
If you did it right, then there is no much differeces between VM and normal setup procedure, just adding Lilu/WEG/AppleALC then the Metal/H264/H265 HW/HDMI Audio will work well in most case.

## USB
We use EHC/UHC from qemu now(XHCI(nec/qemu) can't be recognized by MacOS now).USB port mapping(EHC) and USB Power injection(with fake EC ioreg) already included.

# Known Problems/Help
see issues.
