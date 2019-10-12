---
layout: post
title: Pixel Purse Disassembly
date:   2019-10-03 10:00:00 -0700
categories:
---

*The disassembly of a pretty trivial toy got a bit out of hand. :-)*

* [A Tweet in the Morning](#a-tweet-in-the-morning)
* [Disassembling the Whole Thing](#disassembling-the-whole-thing)
* [LED panel P6 32x16 ](#led-panel-p6-32x16)
* [Main PCB](#main-pcb)
* [SPI PROM contents](#spi-prom-contents)
* [HUB75 Capture](#hub75-capture)

# A Tweet in the Morning

It started with this:

![Tweet]({{ "/assets/pixel_purse/tweet.png" | absolute_url }})

A useless gizmo for only $6.16 where one part is worth more than the whole thing?
Irresistable!

The next day this abomination arrived:

![Pixel Purse Plastic]({{ "/assets/pixel_purse/pixel_purse_plastic.jpg" | absolute_url }})

Calling this a purse is a stretch. It's piece of hard plastic that only looks the part. Barely.

But it's all about the LED screen on the other side:

![Pixel Purse Image]({{ "/assets/pixel_purse/pixel_purse_image.jpg" | absolute_url }})

Taking pictures of a scanning LED display with the scanning camera of an iPhone doesn't work very well. In the real
world, your eyes should see a solid image.

The purse comes with a bunch of built-in patterns and animations. You can create new ones through an iPhone
or Android app.

# Disassembling the Whole Thing

There are [teardown videos](https://www.youtube.com/watch?v=CyLCwa2mneY) on Youtube, but it's super simple: just
a matter of removing a bunch of screws.

![Pixel Purse Unscrewed 0]({{ "/assets/pixel_purse/pixel_purse_unscrewed0.jpg" | absolute_url }})

![Pixel Purse Unscrewed 1]({{ "/assets/pixel_purse/pixel_purse_unscrewed1.jpg" | absolute_url }})

So what can we find inside? It's really very simple:

* 4 AA batteries
* A 3-way power switch
* A button to switch between patterns
* a P6 32x16 LED panel
* the main PCB with all the electronics
* audio plug to download images from your phone to the purse

# LED panel P6 32x16 

A P6 panel has an LED pitch of 6mm. Multiplied by the number of LEDs that gives 192mm x 96mm.

![LED Panel PCB]({{ "/assets/pixel_purse/led_panel_pcb.jpg" | absolute_url }})

This panel is the reason why this purse is such a good deal: on Adafruit, 
[it sells for $25](https://www.adafruit.com/product/420). On AliExpress, you should be able
to find one for ~$14 including shipping. 

The panel has a standard HUB75 interface but with some functionality missing: 
you'd expect an OE_ (BLANK) pin and 5 address lines, but this panel lacks OE\_ altogether and there are only 3
address lines.

With 3 address lines you can select 8 rows. With 2 parallel RGB shift register inputs, you end up with 
16 rows, just what you'd expect for a 32x16 panel.

Important: unused OE_ and 2 upper address lines are not floating, but tied to ground.
This means that you need to be careful in connecting this panel to a generic HUB75 driver that might drive
these pins, since you could create into a short-circuit condition!

Unfortunately, the panel only has a HUB75 input connector, but the output connector has been left unpopulated.

If you want to daisy chain multiple panels, you'll have to solder a connector yourself.

# Main PCB

![Annotated Controller PCB Front]({{ "/assets/pixel_purse/annotated_pcb0.jpg" | absolute_url }})

![Annotated Controller PCB Back]({{ "/assets/pixel_purse/annotated_pcb1.jpg" | absolute_url }})

The controller PCB is spartan, with the following major components:

* microcontroller

  The microcontroller is buried under a blob of hardened plastic. Nothing is known about it.

* 4Mbit SPI PROM 

  XinCore F40A-104GIP

* DC/DC converter

  AP3502E - 340kHz, 2A Synchronous DC-DC Buck Converter

* HUB75 connector

* 32.768kHz Xtal

There are connectors to switches, battery power, and to feed a regulated 5V to the LCD panel. But
there's also an unused connector with I2C and 'VPP' labels. There's little doubt that these are used
to program the microcontroller during production.

# SPI PROM contents

I hoped to be able to repurpose the whole board by reprogramming the microcontroller firmware, so
I dumped the traffic of the SPI during operation. 

In the past, wiring up the small IO pins of the SPI PROM would have been enough of a hassle to just
not bother, but with my micro clips (see my review [here](/tools/2018/04/29/micro-chip-rw-clip.html)),
it takes just a few minutes to do that:

![SPI IO Probes]({{ "/assets/pixel_purse/spi_io_probes.jpg" | absolute_url }})

Connect it to a Saleae logic analyzer, and you're good to go!

![SPI and Saleae]({{ "/assets/pixel_purse/spi_saleae.jpg" | absolute_url }})



Unfortunately, the SPI dump does not contain anything that looks like a firmware. When active,
the microcontroller continuously pulls the LED images from PROM and probably sends it directly
to the panel.

This is especially visible when you switch from displaying a very sparse image (lots of black pixels)
to one with almost all pixels turned on.

# HUB75 Capture

I also wanted to see if there was anything special about the signalling on the HUB75 interface.

Looking at another Saleae capture, you can see that there's repeating pattern with an overall period of
15ms. That's the overall screen refresh rate. During this time, the 3 address lines rotate through
all 8 combinations, thus refreshing all the rows of the panel.

![HUB75 Refresh Rate]({{ "/assets/pixel_purse/HUB75_refresh_rate.png" | absolute_url }})

15ms for 8 addresses means 1.875ms per address.

Zooming in to a single address phase, we see the following:

![HUB75 Single Address]({{ "/assets/pixel_purse/HUB75_single_address.png" | absolute_url }})

There are 4 phases (marked red, green, blue and yellow), each double the duration of the previous one.

During each phase, the clock is toggled 32 times: the number needed to shift in all the 1-bit RGB
values for each LED of the address row.

However when you look at the rectangle in cyan, color values are only shifted in during the shortest
phase. During the other phases, zeros are being shifted into the column drivers.

What's happening here is the following: the microcontroller firmware was designed to support 16 color
shades per pixels, hence the 4 phases of increasing duration. However, the pixel purse only supports
images with 8 colors: the primaries R/G/B are either on or off.

Those primaries are assigned to the LSB of the 4-bit color shade. This has a major benefit that
the LEDs are only driven 1/16th of the normal row active time of 1.875ms, thereby saving a considerable
amount of power. Even this low ON duty cycle is sufficient for the pixel purse: it doesn't need the
light output of an LED panel that was design for huge video walls.

To learn more about driving LED panels with a HUB75 interface, you should read 
[this excellent article](https://www.sparkfun.com/sparkx/blog/2650)  on Sparkfun.

# Uploading New Images

I was particularly interested in how images are uploaded through the audio interface. The overal
process is simple: you enter a new image on an iOS or Android app, you insert audio plug in your
phone or iPad, and off you go.

It's a much system that doesn't require any complex protocol like USB and thus very cheap to
implement.

# DC/DC converter

The whole device is powered by 4 AA batteries, good for a maximum of ~6.5V going down to ~5V when
the batteries are discharged.

A DC/DC converter generates a constant 5V, which is routed to the panel with some
crude soldering of the wires straight onto the panel power connector.

# Power Consumption

![Peak Power Consumption]({{ "/assets/pixel_purse/peak_current.jpg" | absolute_url }})

I did some crude power measurements by cutting off the batteries, and using the current readout
of my bench power supply.

With the power switch set to ON, but the device in deep sleep (which happens after not pressing
any buttons for 30s), power consumption was essentially zero.

Once activated, and with a voltage set to 6.2V, the device consumes between 30 and 150mA depending
on the image shown. There is no image with all LEDs on white, so the maximum is probably
somwhere around 180mA. Using 4 AA batteries with 2200 mAh capacity each, this should result in a
worst case usage of about about 50 hours, much higher that one would expect.

