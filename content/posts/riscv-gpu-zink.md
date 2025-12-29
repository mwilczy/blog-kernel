+++
title = "Igniting the GPU: From Kernel Plumbing to 3D Rendering on RISC-V"
date = "2025-12-28"
description = "How I enabled the PowerVR GPU on the TH1520 SoC by writing the missing kernel drivers."
draft = false
[taxonomies]
  tags = ["Linux Kernel", "RISC-V", "GPU", "Mesa", "Zink"]
+++

## Introduction: Enabling the Hardware

For years, PowerVR GPUs ubiquitous in the embedded world relied entirely on out of tree vendor drivers (often named `pvrsrvkm`). While source code was provided in Board Support Packages, these drivers were never accepted into the mainline kernel due to their non standard architecture.

That changed when Imagination Technologies [announced their commitment](https://lists.freedesktop.org/archives/mesa-dev/2022-March/225699.html) to an upstream, open source driver. The resulting `drm/imagination` driver has been upstream for some time, but it wasn't usable on RISC-V platforms like the T-HEAD TH1520 (used in the Lichee Pi 4A).

**This marks a significant milestone: with the enablement work described below, the TH1520 becomes the first RISC-V SoC to feature fully mainline, hardware accelerated 3D graphics support.**

This effort has followed a long road of development, generating significant community interest along the way from the [initial driver support discussions](https://www.phoronix.com/news/RISC-V-PowerVR-Driver-Support) to the [power sequencing challenges](https://www.phoronix.com/news/T-HEAD-GPU-Power-Sequence), and finally culminating in the [official upstream merge in Linux 6.18](https://www.phoronix.com/news/Linux-6.18-PowerVR-RISC-V).

While the GPU driver itself is generic, the hardware surrounding the GPU on this SoC specifically the power, clock, and reset controllers required significant enablement work before the GPU could actually be probed.

This post details the architectural "plumbing" required to bring up the full graphics stack on the TH1520. This involved implementing the necessary platform drivers to handle the SoC's power sequencing, enabling the mainline `drm/imagination` driver for RISC-V, and validating the stack with a modern, Vulkan based userspace.

---

## Part 1: The Dependency Chain

Enabling the GPU wasn't just a matter of changing a Kconfig entry. The TH1520 GPU subsystem is gated behind a chain of hardware dependencies that had no existing Linux drivers.

To reach the point where I could submit the final patch [enabling the PowerVR driver for RISC-V](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6b53cf48d9339c75fa51927b0a67d8a6751066bd), I first had to implement and upstream the drivers for these underlying subsystems.

The hierarchy looks like this, from the bottom up:

1.  **Mailbox ([`mailbox-th1520`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5d4d263e1c6b6b18acb4d67fd3b9af71b7404924))**: The SoC uses a safety coprocessor (E902) to manage power. The first step was writing a mailbox driver to establish a physical communication link between the main CPUs and this coprocessor.
2.  **Firmware Protocol ([`thead-aon-protocol`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e4b3cbd840e565484d0ad8d260d27c057466ed17))**: On top of the mailbox, I implemented the AON (Always-On) firmware protocol. This driver handles the specific message format required to request power state changes from the coprocessor.
3.  **Power Domains ([`pmdomain-thead`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dc9a897dbb03dfd46c3bd2ce69e42e378bd12ca0))**: With the protocol active, I could expose the GPU's power rail as a standard Linux Generic Power Domain (GenPD). This allows the kernel to manage the GPU's power state generically.
4.  **Resets and Clocks**: Finally, I extended the clock driver ([`clk-th1520-vo`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=50d4b157fa96bfeb4f383d7dad80f8bdef0d1d2a)) and implemented a new reset controller ([`reset-th1520`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4a65326311aba694faafcef9e3c0ef7ae1b722e6)) to handle the specific requirements of the Video Output (VO) subsystem where the GPU resides.

### The Power Sequencer: A Novel Application

With the platform drivers in place, one integration challenge remained. The TH1520 requires a specific, time sensitive sequence to power up the GPU: enable the power domain, wait for voltage stabilization, and then de-assert resets in a specific order.

Historically, power sequencing in the kernel was mostly confined to the MMC/Bluetooth subsystems (for toggling GPIOs on WiFi chips). However, the kernel recently introduced a generic **Power Sequencing (`pwrseq`)** subsystem (authored by Bartosz Golaszewski) to standardize this problem.

During the upstream review process, **Ulf Hansson** (the Power Management subsystem maintainer) suggested that the TH1520's GPU was the perfect candidate for this new framework. It behaves almost like an external component: it needs a dedicated "manager" to orchestrate its wake-up routine before the main driver can even touch it.

I implemented this in [`pwrseq-thead-gpu`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d4c2d9b5b7ceed14a3a835fd969bb0699b9608d3). The most interesting part of this driver is the `match` function, which allows the sequencer to "adopt" the GPU's resources:

{{< highlight c "linenos=true,hl_lines=6-8 19-24" >}}
static int pwrseq_thead_gpu_match(struct pwrseq_device *pwrseq,
				  struct device *dev)
{
	struct pwrseq_thead_gpu_ctx *ctx = pwrseq_device_get_drvdata(pwrseq);

	/* 1. We only match the specific T-HEAD TH1520 GPU compatible */
	if (!of_device_is_compatible(dev->of_node, "thead,th1520-gpu"))
		return 0;

	/*  validation omitted  */

	/* 2. Dynamically acquire resources FROM the consumer device node */
	ctx->num_clks = ARRAY_SIZE(clk_names);
	ctx->clks = kcalloc(ctx->num_clks, sizeof(*ctx->clks), GFP_KERNEL);

	for (i = 0; i < ctx->num_clks; i++)
		ctx->clks[i].id = clk_names[i];

	/* The sequencer grabs the 'core' and 'sys' clocks defined in the GPU's DT node */
	ret = clk_bulk_get(dev, ctx->num_clks, ctx->clks);
	if (ret)
		goto err_free_clks;

	ctx->gpu_reset = reset_control_get_shared(dev, NULL);

	/* ... */
	return 1;
}
{{< /highlight >}}

#### The Integration in `drm/imagination`

To make this work, I also had to introduce a small but strategic change to the generic `drm/imagination` driver (see commit [`e38e8391f30b`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e38e8391f30b41c5a24bb46dc6ef4161921e782d)).

Following a suggestion from **Matt Coster**, I implemented a new abstraction, `pvr_power_sequence_ops`. This interface allows the driver to select its power strategy at runtime based on the device compatible string, keeping the core driver logic generic while accommodating platform specific needs.

For the TH1520, the driver simply selects the `pwrseq` backend:

{{< highlight c "linenos=true" >}}
static const struct pvr_device_data pvr_device_data_pwrseq = {
    .pwr_ops = &pvr_power_sequence_ops_pwrseq,
};

static const struct of_device_id dt_match[] = {
    {
        .compatible = "thead,th1520-gpu",
        .data = &pvr_device_data_pwrseq,
    },
    /* ... */
};
{{< /highlight >}}

This architecture offers three major benefits:

1.  **Clean Abstraction:** The GPU driver doesn't need to know about T-HEAD's specific reset order or microsecond delays. It simply calls the generic `pwr_ops->power_on()`.
2.  **Inversion of Control:** The sequencer "steals" the resource handles (clocks and resets) from the GPU's device tree node during the match phase (lines 19-24 in the first snippet). This allows the sequencer to control resources that conceptually belong to the GPU, ensuring the correct power up order without modifying the GPU driver logic.
3.  **Strict Ordering:** By centralizing this logic in a dedicated driver, we guarantee that the `clkgen` reset (controlled by the parent node) and the `gpu_core` reset (controlled by the consumer node) are de-asserted in the exact order required by the hardware manual.
---

## Part 2: The Display Pipeline (Connecting the Pixels)

Powering up the GPU is a massive victory, but it solves only half the problem. A GPU can render beautiful 3D scenes into memory, but without a **Display Controller** to scan those buffers out to a screen, you're still looking at a black terminal.

On the TH1520, the display duties are handled by a **Verisilicon DC8200** IP block, connected to a Synopsys DesignWare HDMI bridge.

> **Ecosystem Note:** If you are following the RISC-V space, this IP might sound familiar. The **StarFive JH7110** (used in the VisionFive 2) uses the exact same Verisilicon DC8200 display controller.
>
> I am actually working on enabling the display stack for the JH7110 in [parallel](https://lore.kernel.org/all/20251108-jh7110-clean-send-v1-0-06bf43bb76b1@samsung.com/). While the IP is the same, the integration is vastly different the JH7110 has a complex circular dependency between the HDMI PHY and the clock generator that requires a complete architectural rethink. But that is a story for a future blog post.

### The Collaborative Puzzle

While I focused on the TH1520 power sequencing and GPU enablement, the display driver work here was led by **Icenowy Zheng**, another brilliant engineer in the RISC-V ecosystem.

This is the beauty of upstream kernel development: you don't have to build the world alone. Icenowy has been working on a generic DRM driver for [Verisilicon display controllers](https://lore.kernel.org/all/20251224161205.1132149-1-zhengxingda@iscas.ac.cn/), adapting it to support the specific HDMI PHY found on the TH1520.

Since these patches are currently in the review process (v4), they aren't in mainline yet. To build the working demo, I applied Icenowy's patch series on top of mainline kernel.

With Icenowy's display driver handling the "scan out" and my infrastructure handling the "power up," we finally had a complete pipeline: **Memory -> GPU Render -> Memory -> Display Controller -> HDMI**.

## Part 3: The "Vulkan-Only" Future

Now that the kernel could talk to the hardware, we needed a userspace stack to render graphics.

Historically, enabling a new GPU meant writing two massive drivers for Mesa: one for Vulkan and one for OpenGL. But the open-source graphics world has shifted. The `drm/imagination` driver is designed to be **Vulkan-native**.

Instead of writing a complex, legacy OpenGL driver, we use **Zink**.

### The Stack: Rendering vs. Display

Since the TH1520 uses a split DRM architecture, the flow isn't just a straight line. The GPU and Display Controller are separate devices that share data via memory (DMA-BUF).

{{< highlight text >}}
      [ Application (glmark2) ]
                 │
                 ▼
      [    Zink (OpenGL)      ]
                 │
                 ▼
      [ Mesa PowerVR (Vulkan) ]
                 │
      ┌──────────┴──────────┐
      │     Linux Kernel    │
      ▼                     ▼
[ GPU Driver ]       [ Display Driver ]
 (Render Node)         (KMS/Card Node)
      │                     │
      ▼        DMA-BUF      ▼
 [ GPU HW ] ──(Memory)──▶ [ Display HW ] ──▶ HDMI
{{< /highlight >}}

This separation is why the kernel plumbing in Part 1 (GPU) and Part 2 (Display) had to be done independently before they could work together.

### Building the Stack (Reproduction Guide)

For those who want to reproduce this on their own Lichee Pi 4A, exact version matching is critical.

**1. The Kernel**
I used Linux 6.19 as the base, with unmerged Display Controller patches applied on top. You can find the exact tree here:
* **Kernel Branch:** [`github.com/mwilczy/linux/tree/blog_code`](https://github.com/mwilczy/linux/tree/blog_code)

**2. Mesa (Userspace)**
I used a fork of Icenowy Zheng’s work, which includes the necessary glue to make Zink play nicely with this specific hardware combination.
* **Mesa Branch:** [`github.com/mwilczy/mesa`](https://github.com/mwilczy/mesa)

Here is the exact Meson configuration I used to build a pure Vulkan+Zink stack:

{{< highlight bash >}}
meson setup build \
    -D buildtype=release \
    -D platforms=x11,wayland \
    -D vulkan-drivers=imagination \
    -D gallium-drivers=zink \
    -D glx=disabled \
    -D gles1=disabled \
    -D gles2=enabled \
    -D egl=enabled \
    -D tools=imagination \
    -D glvnd=disabled
{{< /highlight >}}

---

## Part 4: The Result

With the kernel compiled (including the pending display patches) and the Mesa stack built, we can finally run accelerated 3D workloads.

### The "Secret Sauce" (Environment Variables)

Because the driver is still in active development and not yet fully conformant, we need to pass a few flags to convince Mesa to run.

The most important one is `PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1`. Without this, the driver safeguards would prevent loading. We also force the use of the Zink driver and explicitly select our device:

{{< highlight bash >}}
export PVR_I_WANT_A_BROKEN_VULKAN_DRIVER=1
export GALLIUM_DRIVER=zink
export MESA_VK_DEVICE_SELECT=1010:36104182!
{{< /highlight >}}

### The Benchmark

I started a Weston compositor session using the DRM backend:

{{< highlight bash >}}
weston --backend=drm-backend.so --continue-without-input &
{{< /highlight >}}

And then, the moment of truth - running `glmark2-es2-wayland`:

![glmark2 running on RISC-V](/images/glmark.png)
*Above: glmark2 running on the Lichee Pi 4A.*

Here is the output, confirming we are running fully accelerated on the PowerVR GPU via Zink:

{{< highlight text>}}
root@revyos-lpi4a:~/test_mesa/Vulkan/build4/bin# glmark2-es2-wayland
MESA: warning: Core count fetching is unimplemented. Setting 1 for now.
WARNING: powervr is not a conformant Vulkan implementation, testing use only.
=======================================================
    glmark2 2023.01
=======================================================
    OpenGL Information
    GL_VENDOR:      Mesa
    GL_RENDERER:    zink Vulkan 1.2(PowerVR B-Series BXM-4-64 MC1 (IMAGINATION_OPEN_SOURCE_MESA))
    GL_VERSION:     OpenGL ES 2.0 Mesa 26.0.0-devel (git-601d20e81e)
    Surface Config: buf=32 r=8 g=8 b=8 a=8 depth=24 stencil=0 samples=0
    Surface Size:   800x600 windowed
=======================================================
[build] use-vbo=false:[  510.861554] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 524288 bytes), total 32768 (slots), used 4 (slots)
[  510.887018] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 1847296 bytes), total 32768 (slots), used 36 (slots)
[  510.900771] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 1847296 bytes), total 32768 (slots), used 36 (slots)
[  510.923956] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 1921024 bytes), total 32768 (slots), used 0 (slots)
[  510.954656] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 458752 bytes), total 32768 (slots), used 0 (slots)
[  510.966931] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 458752 bytes), total 32768 (slots), used 0 (slots)
[  511.038586] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 1921024 bytes), total 32768 (slots), used 0 (slots)
[  512.165193] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 1900544 bytes), total 32768 (slots), used 10 (slots)
[  512.187871] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 1900544 bytes), total 32768 (slots), used 10 (slots)
[  512.209524] verisilicon-dc ffef600000.display: swiotlb buffer is full (sz: 745472 bytes), total 32768 (slots), used 0 (slots)
 FPS: 67 FrameTime: 14.960 ms
[build] use-vbo=true: FPS: 98 FrameTime: 10.252 ms
[texture] texture-filter=nearest: FPS: 97 FrameTime: 10.332 ms
[texture] texture-filter=linear: FPS: 93 FrameTime: 10.868 ms
[texture] texture-filter=mipmap: FPS: 101 FrameTime: 9.957 ms
[shading] shading=gouraud: FPS: 93 FrameTime: 10.851 ms
[shading] shading=blinn-phong-inf: FPS: 98 FrameTime: 10.274 ms
[shading] shading=phong: FPS: 97 FrameTime: 10.356 ms
[shading] shading=cel: FPS: 91 FrameTime: 11.086 ms
[bump] bump-render=high-poly: FPS: 75 FrameTime: 13.404 ms
[bump] bump-render=normals: FPS: 97 FrameTime: 10.356 ms
[bump] bump-render=height: FPS: 88 FrameTime: 11.449 ms
[effect2d] kernel=0,1,0;1,-4,1;0,1,0;: FPS: 94 FrameTime: 10.646 ms
[effect2d] kernel=1,1,1,1,1;1,1,1,1,1;1,1,1,1,1;: FPS: 57 FrameTime: 17.590 ms
[pulsar] light=false:quads=5:texture=false: FPS: 92 FrameTime: 10.953 ms
[desktop] blur-radius=5:effect=blur:passes=1:separable=true:windows=4: FPS: 10 FrameTime: 105.957 ms
[desktop] effect=shadow:windows=4: FPS: 37 FrameTime: 27.465 ms

...
{{< /highlight >}}

We have successfully turned "dark silicon" into a modern, Vulkan capable graphics platform.

---

## Acknowledgements

Bringing a new GPU architecture to life in the mainline kernel is never a solo effort. It requires navigating complex subsystems - from power domains to clocks and relies heavily on the patience and expertise of subsystem maintainers.

This work went through many iterations, and the code is significantly better thanks to the rigorous feedback from the community.

A huge thank you to everyone who helped review the code, suggested architectural improvements, and tested the stack:

* **Marek Szyprowski** - For the guidance and mentorship throughout the upstreaming process.
* **Drew Fustini** - For his long standing work maintaining the TH1520 platform.
* **Krzysztof Kozłowski** - For ensuring the Device Tree bindings were strictly compliant.
* **Ulf Hansson** - For his guidance on the AON power domains and for suggesting the use of the Power Sequencing framework, which simplified the architecture significantly.
* **Bartosz Gołaszewski** -  For creating the Power Sequencing subsystem and helping merge the TH1520 driver.
* **Matt Coster** - For reviewing the driver changes and helping navigate the PowerVR internals.
* **Stephen Boyd**  - For the feedback on the video output clock controller.
* **Philipp Zabel** - For reviewing the reset controller implementation.
* **Icenowy Zheng** - For the incredible work on the display controller and Mesa/Zink integration.
* **Jassi Brar** - For reviewing the mailbox driver implementation.
* **Conor Dooley** - For reviewing Device Tree patches.
