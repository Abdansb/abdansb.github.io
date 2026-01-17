---
title:  "External VRM Mod Project Radeon HD6670"
date:   2025-11-01 12:34:05 +0700
categories: [Hardware]
---
A hardware modification project to unlock the performance of an aging GPU by providing stable, adjustable voltage via an external Voltage Regulator Module (VRM).

![Radeon 6670 chip codename Turks XT](/assets/img/radeon/turks.webp)

## Introduction

The MSI Radeon HD 6670 1GB GDDR5 is a mid-range graphics card from AMD's Northern Islands generation. Key specifications relevant to this project include:

- GPU Core: Turks XT (40nm process)
- Stream Processors: 480
- Core Clock Speed: 800 MHz (stock)
- Memory Clock Speed: 1000 MHz (4.0 Gbps GDDR5 effective)
- Memory Interface: 128-bit
- TDP (Thermal Design Power): ~66W
- Power Connector: No external PCIe power connector (relies solely on PCIe slot power, 75W max)
- Cooling Solution: Single fan, heatsink assembly.
- VRM Configuration: Internal, typically 3 phase design for GPU and memory power delivery

These specs show how much the card relies on the PCIe slot for power, which limits how much i can overclock and boost performance. That's why an external VRM mod is a pretty good idea. Also, the stock cooler is made for the 66W TDP, so i'll need to keep an eye on it if i bump up the power.

### 1. Motivation

![VRM Driver or PWM Controller IC](/assets/img/radeon/up6201B.webp)

Why an External VRM?

The power system on my MSI Radeon HD 6670 totally crapped out. Turns out the stock PWM controller, that little uP6201B chip, shorted. That's the part that's supposed to control the VRM MOSFETs and deliver the Vcore voltage to the GPU, so without it, the graphics card is basically a fancy paperweight.

Instead of trying to replace that tiny, surface-mounted chip – which would normally need a hot air station, and I only have a basic soldering iron – I decided to go a different route. I actually have another broken GPU with a perfectly good VRM section on it. So, the plan is to harvest that working VRM and integrate it with the MSI Radeon HD 6670. This way, I'm not just fixing the card. I'm also giving it a huge upgrade. The new VRM lets me control the voltage manually using hardware. This opens the door for real overclocking, something the stock card could never do.

### 2. Understanding the MSI Radeon HD 6670

#### 2.1 The Voltage Regulator Modules (VRMs)

![VRM Driver or PWM Controller IC](/assets/img/radeon/up6201B_diagram.PNG)

**Figure 1**: The official datasheet schematic for the uP6201B, a 2-phase synchronous buck controller which manages the GPU's core voltage.

The GPU's core voltage is supplied by a Voltage Regulator Module (VRM) that uses a 2-phase synchronous buck converter, managed by the uP6201B controller shown in Figure 1. A buck converter's primary function is to step down a higher input voltage (e.g., 12V) to a much lower, precise output voltage (e.g., 1.0V for the GPU core).

#### 2.2 Schematics Diagram
In a 2-phase design, the total power delivery is split between two identical circuits, or "phases," which operate interleaved (180 degrees out of sync). This distributes heat, reduces electrical noise, and allows for a faster response to sudden changes in GPU load.

![VRM Driver or PWM Controller IC](/assets/img/radeon/Ugate.PNG)

**Figure 2:** A 2-phase VRM reference schematic from a Radeon 6770, which is functionally identical to the circuit on the HD 6670\.

To understand how the uP6201B is implemented, we can analyze the reference schematic. Although this diagram is from a Radeon 6770, the circuit topology and component numbering (e.g., Q801, L801) are virtually identical to the HD 6670 board.

This diagram illustrates a discrete 2-phase design. **PHASE1** is designed with one high-side MOSFET (Q801) and has pads for two parallel low-side MOSFETs (Q802 and Q803). Using two low-side FETs is a common technique to handle higher current and distribute heat on more power-hungry GPUs. However, Q803 and Q813 are often optional. **PHASE2** mirrors this design (Q811, Q812, Q813, and L811). The controller sends signals (like `UGATE1_CTR` from the `uP6201B`) to the gates of these MOSFETs, switching them on and off thousands of times per second to produce the final `+VDDC` (GPU core voltage).

![VRM Driver or PWM Controller IC](/assets/img/radeon/Mosfet.webp)

**Figure 3:** The physical 2-phase VRM circuit on the MSI Radeon HD 6670 PCB, showing a more integrated implementation.

On the actual MSI HD 6670 PCB, as shown in **Figure 3**, we see a more cost-effective and integrated version of this circuit. The pads for the secondary low-side MOSFETs (Q803, Q813 from the schematic) are not populated, as they are unnecessary for this card's power target.

The "Choke Inductors" (L801/L811) are the next component in the power path.

These inductors are critical. Beyond storing energy, their internal resistance (DCR) is utilized by the uP6201B's **current sensing** circuit to monitor the load. This current sense data, along with the main **voltage feedback** loop, is essential for the controller to operate correctly.

This inductors on the PCB are the connection points for the donor VRM. By cutting the traces immediately after these inductors and their associated output filter capacitors, the GPU core will be isolated from the original, faulty VRM. This allows the newly integrated external VRM to deliver power directly to the GPU, utilizing the existing filtering network while bypassing the defunct onboard power delivery system. This way ensures that the GPU receives clean, regulated voltage from the donor VRM without interference from the original, damaged circuit.

#### 3. The Donor VRM From Asus Radeon R7 240 Low Profile

![VRM Driver or PWM Controller IC](/assets/img/radeon/R7-240.jpg)

**Figure 4:** The donor card, an Asus Radeon R7 240 Low Profile (image courtesy of TechPowerUp).

For this project, the replacement voltage regulator module will be salvaged from a donor card, an Asus Radeon R7 240\. This card was acquired in a non-functional state (due to a faulty GPU core) from a local marketplace for approximately $2 USD. While the GPU core is dead, its VRM section was tested and confirmed to be fully operational.

This card's VRM design is fundamentally different from the target HD 6670\. The **MSI HD 6670** uses **one** controller IC (uP6201B) that is responsible for driving **two** phases simultaneously. In contrast, the **Asus R7 240** uses **two** separate controller ICs (EM5305)—one for each phase.

#### 3.1 Typical Application Circuit For The Controller

![VRM Driver or PWM Controller IC](/assets/img/radeon/EM5305_diagram.PNG)

**Figure 5:** The typical application circuit for the EM5305, synchronous buck controller used on the R7 240\.

The EM5305 is a dedicated **single-phase synchronous buck controller**.

Similar to the uP6201B, its job is to drive external MOSFETs. As shown in the diagram, the chip sends switching signals from its `UGATE` (Upper Gate) and `LGATE` (Lower Gate) pins to a pair of external N-Channel MOSFETs. These MOSFETs then feed the main power inductor and filter capacitors. The chip continuously monitors the output voltage via the **Feedback (FB)** pin and adjusts the switching of the MOSFETs to maintain a perfectly stable voltage.

The R7 240's VRM simply consists of *two complete copies* of this circuit, side-by-side, working in parallel to share the load. This design choice was likely made due to the card's **low-profile form factor**, as using two smaller, single-phase controllers allows for more flexible component placement on a cramped PCB.

Honestly, this two separate IC design offers significant advantages for repair and modification. It is much easier to troubleshoot because each phase is independent. If one phase fails, the fault can be isolated to a single EM5305 IC and its paired MOSFETs. The reduced pin count of the IC (as seen on the EM5305) also simplifies the probing and diagnosis process.

For my modification, I plan to physically cut the traces *after* the inductors and output filter capacitors to eliminate the rest of PCB. This step is important because the IC is designed to protect itself; if I cut the trace before the inductors, the IC will not output voltage. If the controller does not sense any current flowing from its circuit, it will register a fault condition (like an open circuit) and will shut down, outputting no voltage.

### 3.1 Hardware Components

Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
