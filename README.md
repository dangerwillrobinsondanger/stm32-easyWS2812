# stm32-easyWS2812

A simple WS2812 driver for STM32 Microcontrollers

# Theory of Operation

This is actually less a "driver" than a utility library; all of the work of sending out the bitstream to an arbitrary number of WS2812 LEDs is done by an STM32 general purpose timer and a DMA channel.

The data format for WS2812 LEDs is well-documented in the part's [Datasheet](https://cdn-shop.adafruit.com/datasheets/WS2812.pdf). It consists of a reset pulse (50μsec or longer low pulse) followed by a series of pulse-width-encoded bits. Strings of LEDs are configured by sending repeating Green-Red-Blue 24-bit (3 colors x 8 bits per color) sequences; each LED repeats received sequences to downstream LEDs and all LEDs latch and display the **last** 24-bit color sequence they receive.

![WS2812 Bit Values](https://cpldcpu.files.wordpress.com/2014/01/ws2812_protocol.png)

This essentially requires wiggling an output pin with a bunch of pulses whose width represents 1s or 0s. We could do this with a software loop (as the WS2812 Arduino class does) or using DMA directly to a GPIO register. The former approach is CPU intensive, and the latter requires addressing an entire port's 16-bit output value register as the DMA target. This is fine if every pin of the port is connected to a string of WS2812 LEDs, but is far less memory efficient and commandeers otherwise-useful pins.

This setup is similar to the [Daedaluz library](https://github.com/Daedaluz/stm32-ws2812), except with documentation and without the use of Ethernet for demonstration. We configure a timer to roll-over (generating a DMA event) at the desired bit period (1.2μsec). We configure the timer's output channel in PWM Generation mode. The value of the output channel's compare register will determine the width of the generated pulse.

We then configure a DMA channel to load a new output compare register value each time the counter rolls over. The souce is a RAM buffer containing the compare register values corresponding to "0" or "1" pulse widths. The RAM buffer is pre-loaded with the requisite reset pulse (> 50μsec).

The ws2812_write() function just fills the RAM buffer with the appropriate pulse width values. The RAM buffer must be sized according to the number of WS2812 LEDs in the string; preprocessor defines are provided for this.

# Setup

The src/ directory of this repository includes a complete example [System Workbench for STM32](http://www.st.com/en/development-tools/sw4stm32.html) (SW4STM32) Eclipse project, ready to be imported and built. The project is generated via the [STM32CubeMx](http://www.st.com/en/development-tools/stm32cubemx.html) configuration tool. The example project targets a [STM32F051R8](http://www.st.com/en/microcontrollers/stm32f051r8.html) microcontroller, readily available via the [STM32F0DISCOVERY](http://www.st.com/content/st_com/en/products/evaluation-tools/product-evaluation-tools/mcu-eval-tools/stm32-mcu-eval-tools/stm32-mcu-discovery-kits/stm32f0discovery.html) development kit.

![STM32F0DISCOVERY](http://www.st.com/content/ccc/fragment/product_related/rpn_information/board_photo/68/3f/1b/af/87/47/4a/91/stm32f0discovery.jpg/files/stm32f0discovery.jpg/_jcr_content/translations/en.stm32f0discovery.jpg)

The example application uses PA0 as the WS2812 bitstream output, with the output waveform generated by Timer 2 Channel 1.

# Usage

To incorporate the stm32-easyWS2812 utility into a new project, add the ws2812.h and ws2812.c files. If using STM32CubeMx to generate a HAL, configure the peripherals as follows:

* Configure the MCU clock tree to provide sufficiently-fast SYSCLK and APB rates to allow reasonably fine-grained control over the duty cycle of the PWM output waveform. (In the example, I fed the PLL from the HSI at 8MHz, producing a 32MHz system clock. The AHB prescaler is set to 1, keeping the AHB clock at 32MHz as well.)
* Configure a Timer to roll over at a 1200ns interval. (Example: With AHB clock at 32MHz and no prescaling, Timer 2 requires an auto reload register value of 39 to produce a 1218ns period.)
* Configure an output channel of the timer in PWM generation mode, with the output brought out to a GPIO pin in alternate-function mode. (In the example, I use PA0.)
* Configure a DMA channel in memory-to-peripheral mode, using the timer channel's DMA request as the trigger.

In your user code, you need only call a single HAL API function to set up the DMA source, target and trigger and start the timer running:

```
if(HAL_OK != HAL_TIM_PWM_Start_DMA(&htim2, TIM_CHANNEL_1, pixel_buffer, PIX_BUF_SIZE))
{
	for(;;) { /* Trap */ }
}
```

Once the timer and DMA channel are running, no further input is required from the software to continually refresh the string of WS2812 LEDs from the RAM pixel buffer. The only further activity required from the software is to call the `ws2812_setpix()` function whenever the colors in the RAM pixel buffer need to be changed.

# About the Example

The example application provided here is a sunrise simulator, meant for use as an ambient light alarm clock. On startup, the application begins illuminating a strip of WS2812 LEDs that could be tucked behind a dresser or headboard.

The lighting patterns are taken from time-lapse videos of sunrises, and the example application runs the simulation over 30 seconds (rather than the 30 minutes it would normally occupy) so that the progression from violet and blue, through deep reds and organges, through bright yellows and concluding with white light is easily visualized.
