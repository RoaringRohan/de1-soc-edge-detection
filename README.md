# DE1-SoC Edge Detection

Canny edge detection on a DE1-SoC, implemented twice: once in software on the ARM core, and once by
streaming the image through a hardware edge detector in the FPGA fabric.

That pairing is the point of the project. The same algorithm on the same board, on the same images,
with the work moved across the hardware/software boundary — which is the question a chip with a CPU
and an FPGA on one die exists to ask.

> **Demo video:** *(to be added)*
>
> This needs the physical board, so there is no capture in this README yet — see
> [Why there is no demo here](#why-there-is-no-demo-here).

## The algorithm

Canny is four stages after the image is reduced to greyscale, and each is implemented directly:

**1. Gaussian blur.** A 5×5 convolution against a fixed integer kernel, with the weighted sum
divided by 273 — the sum of the kernel weights — so the result stays in range without floating
point. Out-of-bounds neighbours are treated as zero rather than clamped or mirrored. Smoothing first
matters because the next stage differentiates, and differentiation amplifies noise.

**2. Sobel filter.** Two 3×3 convolutions giving the horizontal and vertical intensity gradients,
kept as signed integers so direction survives.

**3. Non-maximum suppression.** The gradient magnitude is thinned to single-pixel-wide ridges by
comparing each pixel against its two neighbours along the gradient direction and discarding it
unless it is the local maximum. Without this an edge comes out as a wide band rather than a line.

**4. Hysteresis.** A pixel survives only if it exceeds the intensity threshold *and* has at least
one of its eight neighbours also above it. Textbook Canny uses two thresholds — a high one to seed
edges and a low one to extend them — where this uses a single threshold plus a connectivity
requirement, which is the simplified form the exercise specifies. It still does the job that
matters: isolated bright pixels are dropped while connected edge fragments survive.

Running with `-d` writes each stage to its own BMP, so the intermediate images can be inspected
rather than taken on trust.

## Moving it into hardware

The FPGA is then reconfigured with a different design — an edge detector built as a **streaming
pipeline** in fabric — and the same work is done by feeding the image through it.

The interface is DMA, not memory-mapped registers, because the data is an image rather than a value:

| Part | What runs where |
|---|---|
| **1** | Everything on the ARM core. Pure software Canny, four stages, one image at a time |
| **2** | FPGA reconfigured; a **memory-to-stream DMA** at lightweight-bridge offset `0x3100` pushes pixels into the hardware pipeline |
| **3** | Adds the return path — a **stream-to-memory DMA** at offset `0x3120` — so results come back to SDRAM and the full round trip runs in hardware |

The image buffer lives in SDRAM, mapped through the non-lightweight HPS-to-FPGA bridge, because the
FPGA on-chip memory is too small to hold a 640×480 image. That is why parts 2 and 3 map two regions
rather than one.

Reconfiguring the FPGA at runtime is itself a constraint: every driver that touches the fabric has
to be unloaded first, or it will be holding addresses that are about to stop meaning what they
meant. `removeModulesAndConfigureFPGA.sh` unloads the video driver before loading the new bitstream,
in that order, for that reason.

## On performance

The software and hardware paths are both instrumented — `clock()` around the pipeline, printing
elapsed milliseconds — so the comparison can be measured directly when the code runs:

```
TIME ELAPSED: %.0f ms
```

**No measured figure is recorded in this repository.** The timing is printed at runtime and was
never captured into a results file, so this README quotes no speedup, and any number attached to
this project elsewhere should be treated as unverified until it is re-measured on the board. The
instrumentation is there; the result is not.

## Building and running

On a DE1-SoC running the course Linux image:

```bash
cd part1
./runModules.sh                  # loads the video driver
make
./edgedetect bridge_640_480.bmp -d -v    # -d writes each stage, -v draws to the display

cd ../part3
./removeModulesAndConfigureFPGA.sh       # unload drivers, then load the edge-detector bitstream
make
./edgedetect bridge_640_480.bmp
```

Input images and the FPGA bitstreams are **not** in this repository — see below.

## Why there is no demo here

This is FPGA and embedded work: it reconfigures the fabric at runtime and moves image data over DMA
between an ARM core and a streaming pipeline at fixed physical addresses. It cannot run anywhere
else, and a container does not help. Nothing here was executed during publication, and there is no
screenshot — the only honest one would be a photograph of a monitor, or an output BMP produced on
hardware that was not available.

A demo video is planned and will be linked at the top of this README when it exists.

## A note on what is here, and it matters more on this project than its siblings

**The course template was substantial on this exercise, and the honest accounting is specific.** It
supplied 695 lines across the three parts — BMP reading and writing, greyscale conversion, the
display path, `main`, argument handling and the stage orchestration — against 939 lines published.

What was written is the part the template marked `please complete this function`, four times over:
**`gaussian_blur`, `sobel_filter`, `non_max_suppress` and `hysteresis_filter`** — the four stages of
the algorithm. The plumbing around them was given. That is a smaller line count than the file
listing suggests and a more interesting contribution than it sounds: BMP parsing is verbose and the
convolutions are dense, so the ratio understates the work rather than overstating it. Parts 2 and 3
additionally required carrying the image I/O forward and wiring up the DMA controllers.

Also course-supplied and **deliberately not republished**: the two FPGA bitstreams
(`DE1_SoC_Computer.rbf` and `Edge_Detector_System.rbf`, compiled binaries totalling several
megabytes), the thirteen test images, and the lab handout. The figures in the project notes are
reproduced from the DE1-SoC Computer manual and are likewise not included; the hardware details
above are written out in original prose with the manual credited.

Included because the code does not build without them: `include/address_map_arm.h`,
`include/physical.h`, `physical.c` and the per-part `Makefile`s.

This is one of four projects published from the same course. It shares the DE1-SoC address map and
the `/dev/mem` helpers with
[de1-soc-vga-renderer](https://github.com/RoaringRohan/de1-soc-vga-renderer),
[de1-soc-piano](https://github.com/RoaringRohan/de1-soc-piano) and
[de1-soc-oscilloscope](https://github.com/RoaringRohan/de1-soc-oscilloscope) — that header is
reference material, not original to any of them.

Built with a partner.

---

*Originally built as a graduate course project.*
