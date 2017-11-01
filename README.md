# stm32-easyWS2812

A simple WS2812 driver for STM32 Microcontrollers

# Theory of Operation

This is actually less a "driver" than a utility library; all of the work of sending out the
bitstream to an arbitrary number of WS2812 LEDs is done by an STM32 general purpose timer and a DMA
channel.

The data format for WS2812 LEDs is well-documented in the part's
[Datasheet](https://cdn-shop.adafruit.com/datasheets/WS2812.pdf). It consists of a reset pulse
(50μsec or longer low pulse) followed by a series of pulse-width-encoded bits. Strings of LEDs are
configured by sending repeating Green-Red-Blue 24-bit (3 colors x 8 bits per color) sequences; each
LED repeats received sequences to downstream LEDs and all LEDs latch and display the **last** 24-bit
color sequence they receive.

![WS2812 Bit Values](https://cpldcpu.files.wordpress.com/2014/01/ws2812_protocol.png)

This essentially requires wiggling an output pin with a bunch of pulses whose width represents 1s or
0s. We could do this with a software loop (as the WS2812 Arduino class does) or using DMA directly
to a GPIO register. The former approach is CPU intensive, and the latter requires addressing an
entire port's 16-bit output value register as the DMA target. This is fine if every pin of the port
is connected to a string of WS2812 LEDs, but is far less memory efficient and commandeers
otherwise-useful pins otherwise.

This setup is similar to the [Daedaluz library](https://github.com/Daedaluz/stm32-ws2812), except
with documentation and without the use of Ethernet for demonstration. We configure a timer to 
roll-over (generating a DMA event) at the desired bit period (1.2μsec). We configure a timer output
channel in PWM Generation mode. The value of the output channel's compare register will determine
the width of the generated pulse.

We then configure a DMA channel to load a new output compare register value each time the counter
rolls over. The souce is a RAM buffer containing the compare register values corresponding to "0" or
"1" pulse widths. The RAM buffer is pre-loaded with the requisite reset pulse (> 50μsec).

The ws2812_write() function just fills the RAM buffer with the appropriate pulse width values.

# Setup

TODO

# Usage

TODO
