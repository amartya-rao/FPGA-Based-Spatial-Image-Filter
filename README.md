# SpatialFilter on FPGA

Hardware image processing pipeline for **Xilinx Zynq** that performs streaming **3x3 spatial filtering** on a grayscale `512 x 512` image.

This project contains:

- A custom **AXI-Stream image-processing IP**
- Verilog RTL for line buffering, window generation, and convolution
- A Vivado project with the IP packaged for block-design use
- A bare-metal Zynq software example that moves image data through the IP with **AXI DMA**
- A simulation testbench that reads a BMP image and writes the filtered result back to disk

The default configuration in the current top-level design is a **3x3 box blur / mean filter**. An additional **edge-detection variant** is also included in the repository.

## What The Design Does

The hardware accepts a stream of 8-bit grayscale pixels, builds a sliding `3x3` window, applies a convolution kernel, and streams the processed pixels back out.

High-level dataflow:

```text
Input pixel stream
    -> line buffers
    -> 3x3 pixel window generation
    -> convolution engine
    -> AXI-Stream output buffer
    -> output pixel stream
```

In the Zynq system flow, software running on the Processing System:

- feeds the image into the IP through **AXI DMA**
- responds to the IP interrupt when another row can be sent
- receives the processed output back into memory
- transmits the processed image bytes over UART

## Architecture

### 1. `imageProcessTop`

Top-level streaming module: [rtl/imageProcessTop.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/imageProcessTop.v)

- Exposes AXI-Stream style input and output signals
- Instantiates the control path and convolution block
- Buffers filtered pixels through `outputBuffer`
- Raises `o_intr` to request the next row of image data

### 2. `imageControl`

Window-generation and flow-control module: [rtl/imageControl.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/imageControl.v)

- Uses **four line buffers** to store incoming rows
- Starts producing valid `3x3` windows once enough pixels are available
- Rotates read/write ownership across the line buffers
- Generates an interrupt after a row-sized processing chunk completes

### 3. `lineBuffer`

Single-row storage primitive: [rtl/lineBuffer.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/lineBuffer.v)

- Stores one row of `512` pixels
- Outputs three horizontally adjacent pixels at a time
- Supports the sliding window used by `imageControl`

### 4. `conv`

Default convolution engine: [rtl/conv.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/conv.v)

- Multiplies the `3x3` neighborhood by a kernel
- The current kernel is initialized to all ones
- Divides the accumulated sum by `9`
- Produces a **mean-filtered / blurred** output pixel

Effective kernel:

```text
1 1 1
1 1 1   / 9
1 1 1
```

### 5. `conv1`

Alternative convolution module: [rtl/conv1.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/conv1.v)

- Implements a Sobel-style gradient calculation using two kernels
- Squares and combines horizontal and vertical responses
- Thresholds the result to generate a binary edge map

Note: this file is present in the repository, but the current top-level design wires in `conv.v`, not `conv1.v`.

## Interfaces

The packaged IP in [ip/component.xml](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/ip/component.xml) exposes:

- `s_axis`: AXI-Stream slave input for incoming pixels
- `m_axis`: AXI-Stream master output for processed pixels
- `axi_clk`: streaming clock
- `axi_reset_n`: active-low reset
- `o_intr`: interrupt output to the processor

An internal generated FIFO `outputBuffer` is used on the output side of the pipeline.

## Repository Layout

```text
.
|-- README.md
|-- rtl/
|   |-- imageProcessTop.v
|   |-- imageControl.v
|   |-- lineBuffer.v
|   |-- conv.v
|   `-- conv1.v
|-- tb/
|   `-- tb.v
|-- sw/
|   |-- imageIpTest.c
|   `-- imageData.h
|-- testImage/
|   `-- lena_gray.bmp
`-- ip/
    |-- imageProcessing.xpr
    |-- component.xml
    `-- imageProcessing.srcs/
```

## Simulation Flow

Testbench: [tb/tb.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/tb/tb.v)

The simulation testbench:

- opens `lena_gray.bmp`
- copies the BMP header
- streams pixel bytes into the DUT
- waits for row-level interrupts from the hardware
- writes the filtered result into `blurred_lena.bmp`

Important implementation details:

- Image size is fixed at `512 x 512`
- BMP header size is assumed to be `1080` bytes
- The testbench initially sends `4 x 512` bytes before switching to interrupt-driven row updates
- Two trailing rows of zeros are sent to flush the pipeline

### Running Simulation In Vivado

1. Open [ip/imageProcessing.xpr](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/ip/imageProcessing.xpr) in Vivado.
2. Make sure `lena_gray.bmp` is available in the simulator working directory, or update the file path in [tb/tb.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/tb/tb.v).
3. Run behavioral simulation.
4. Inspect the generated `blurred_lena.bmp`.

## Zynq Software Flow

Software example: [sw/imageIpTest.c](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/sw/imageIpTest.c)

The bare-metal application does the following:

- configures `UART1`
- initializes `AXI DMA`
- sets up the interrupt controller
- starts one DMA receive transfer for the full output image
- sends the first four rows into the image-processing IP
- feeds one additional row each time the IP raises an interrupt
- marks completion when the DMA receive interrupt fires
- transmits the processed image bytes over UART

Image data source: [sw/imageData.h](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/sw/imageData.h)

This header contains the raw grayscale image bytes compiled directly into the software image.

## Current Assumptions And Limits

- Grayscale pixels only: `8 bits` per pixel
- Fixed image size: `512 x 512`
- Streaming operation is row-oriented
- The shipped RTL is set up for a `3x3` filter window
- Border handling is implicit in the buffering / flush behavior rather than parameterized
- The repository is Vivado-centric and includes packaged-IP artifacts

## If You Want To Modify The Filter

For a different spatial filter, the main place to start is [rtl/conv.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/conv.v).

Examples:

- Change the kernel coefficients for sharpening, embossing, or Gaussian-like blur
- Replace `conv.v` with the Sobel-style logic in [rtl/conv1.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/conv1.v)
- Generalize the image dimensions and line-buffer depth
- Add coefficient programmability through registers or AXI-Lite

## Source Files To Start With

- Hardware entry point: [rtl/imageProcessTop.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/imageProcessTop.v)
- Window control: [rtl/imageControl.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/imageControl.v)
- Filter math: [rtl/conv.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/conv.v)
- Edge-detection variant: [rtl/conv1.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/rtl/conv1.v)
- Testbench: [tb/tb.v](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/tb/tb.v)
- Zynq software example: [sw/imageIpTest.c](/Users/amartyaraoponugoti/Image%20Processing%20in%20FPGA/sw/imageIpTest.c)

## Reference Videos

The original README linked a video series for the build:

- Introduction: https://youtu.be/Zm3KzhahbUg
- Design of line buffer: https://youtu.be/n35zS__YEFQ
- Design of MAC: https://youtu.be/6El_NQrpgCY
- Design of control logic: https://youtu.be/v8pHH-q-0sE
- IP packaging: https://youtu.be/v8xZ7ZlJ3ek
- Simulation: https://youtu.be/tO5pZ2K9U9I
- System design: https://youtu.be/r45dkUHIbk4
- Software design: https://youtu.be/IO0hTR3ymdA
- Edge detection: https://youtu.be/TcjqZG2pbHw
