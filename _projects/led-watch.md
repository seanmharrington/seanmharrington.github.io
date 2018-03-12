---
comments: true
tags: [MSP430, Hardware Design, Embedded, LED, Wristwatch]
title: "LED Wristwatch"
classes: wide
header:
  teaser: /assets/images/led-watch-banner-1.1.jpg
toc: true
---

The LED Wristwatch - a challenge for myself that started years back when I decided to step into the hardware design world.

## Background
Back in 2014 it was my 3rd year into my Computer Engineering degree and also 3rd year as a member of the Ohio State University team known as Buckeye Current. At this point, I had a large amount of experience with writing software for many of the custom hardware boards used by the Buckeye Current team. This software ranged from custom CAN bus dataloggers, sensor measurement devices, and driver control systems. I however had an itch to design some hardware as well. I decided it was time I designed my first PCB after watching others do it for years.

Instead of designing a simple, minimal-functionality, two-layer board as my first PCB, I decided to step it up to the next level immediately by designing a largely populated four-layer board that could fit on my wrist. Born was the idea of a LED wristwatch.

## Getting started
I'll note that at the time of starting this project, I had already seen many custom PCBs that my friends were designing and had a decent working list of dos and don'ts when it came to PCB design. After many iterations, I have learned a lot more than when I first started. 

It all started with version 1.0. This version solved the complexity of placing a MCU, 132 LEDs, and all the other necessary components within a PCB diameter less than 1.5". It however came up short due to a critical mistake when laying out the pin assignments on the MCU. For that reason, version 1.0 was never fully assembled before I moved on to version 1.1 to resolve the issues found in 1.0. For a full detailed review on version 1.0, see [LED Watch v1](/led-watch-v1)

## Gen. 2
I took some time off from the project after finding the issues with version 1.0. When I finally came back to this project, I decided to relook at many different design considerations from version 1.0. At this point I was also engaged and figured it would be neat to give these watches to all my groomsmen. It was time to get back to it.

Gen. 2 also known as version 1.1 not only fixed issues with 1.0 but was a complete re-layout of the PCB and included a new fun feature: __Ambient Light Detection__. I moved away from EAGLE and started recreating the work in KiCAD which is open-source. The following changes and features were made in v1.1:
* Recreated PCB and schematic using KiCAD
* Increased board diameter to approximately 1.4". Found existing watch enclosure that the PCB could fit in with this increased size.
* Ambient light detection
* Improved external oscillator design
* Changed debug/programming port to be 0.05" pin header for accessibility
* Change all passives to be 0402 footprint
* Moved all footprints other than battery holder and buttons to top layer to minimize additional soldering needed on PCB bottom after using reflow oven.

This time I created a functional PCB that I assembled completely by hand using just a soldering iron (it didn't take as long as you probably imagine). After finding and patching some mistakes in the PCB design --- it just worked. During the move from EAGLE to KiCAD, I forgot to add a ground via between the crystal capicators and I left out a ground connection that created two isolated grounds on the PCB. Some hookup wire solved the issues with minimal effort.

![Gen 2 Populated](/assets/images/led-watch-v11-populated.jpg "LED Watch v1.1 Populated")

With functional hardware, it was possible to start testing and modifying the software that was to control the LEDs and time. Originally, I had planned that very little memory was required for a clock as the software routine is fairly simple and I therefore went with the MSP430FR5721 that has only 4KB of non-volatile memory. Early on I stuggled to fit the program within the available memory just from the logic needed to control the LED matrix. Even with heavy compiler optimizations enabled, it became challenging to fit a function lookup table for 132 LEDs that simply performed the minimum amount of register writes to toggle a GPIO and light up the correct LED. 

```c
/*
 * led_controller.c
 *
 *  Created on: Aug 21, 2015
 *      Author: Sean
 */

#include <msp430.h>
#include "Headers\led_controller.h"


// LED Lookup Tables
//
// Stores function pointers
//
// LED Seconds Lookup
// 		Index 0-59 		-> Seconds
// LED Minutes Lookup
// 		Index 0-59 		-> Minutes
// LED Hours Lookup
// 		Index 120-131	-> Hours


void(*led_seconds[60])();

void(*led_minutes[60])();

void(*led_hours[24])();

void initialize_leds()
{
	// Intialize LED lookup tables
	led_seconds[0] = zero_seconds;
	led_seconds[1] = one_seconds;
	led_seconds[2] = two_seconds;
	led_seconds[3] = three_seconds;
	led_seconds[4] = four_seconds;
	led_seconds[5] = five_seconds;
	led_seconds[6] = six_seconds;
	led_seconds[7] = seven_seconds;
	led_seconds[8] = eight_seconds;
	led_seconds[9] = nine_seconds;
	led_seconds[10] = ten_seconds;
	led_seconds[11] = eleven_seconds;
	led_seconds[12] = twelve_seconds;
	led_seconds[13] = thirteen_seconds;
	led_seconds[14] = fourteen_seconds;
	led_seconds[15] = fifteen_seconds;
	led_seconds[16] = sixteen_seconds;
	led_seconds[17] = seventeen_seconds;
	led_seconds[18] = eighteen_seconds;
	led_seconds[19] = nineteen_seconds;
	led_seconds[20] = twenty_seconds;
	led_seconds[21] = twentyone_seconds;
	led_seconds[22] = twentytwo_seconds;
	led_seconds[23] = twentythree_seconds;
	led_seconds[24] = twentyfour_seconds;
	led_seconds[25] = twentyfive_seconds;
	led_seconds[26] = twentysix_seconds;
	led_seconds[27] = twentyseven_seconds;
	led_seconds[28] = twentyeight_seconds;
	led_seconds[29] = twentynine_seconds;
	led_seconds[30] = thirty_seconds;
	led_seconds[31] = thirtyone_seconds;
	led_seconds[32] = thirtytwo_seconds;
	led_seconds[33] = thirtythree_seconds;
	led_seconds[34] = thirtyfour_seconds;
	led_seconds[35] = thirtyfive_seconds;
	led_seconds[36] = thirtysix_seconds;
	led_seconds[37] = thirtyseven_seconds;
	led_seconds[38] = thirtyeight_seconds;
	led_seconds[39] = thirtynine_seconds;
	led_seconds[40] = fourty_seconds;
	led_seconds[41] = fourtyone_seconds;
	led_seconds[42] = fourtytwo_seconds;
	led_seconds[43] = fourtythree_seconds;
	led_seconds[44] = fourtyfour_seconds;
	led_seconds[45] = fourtyfive_seconds;
	led_seconds[46] = fourtysix_seconds;
	led_seconds[47] = fourtyseven_seconds;
	led_seconds[48] = fourtyeight_seconds;
	led_seconds[49] = fourtynine_seconds;
	led_seconds[50] = fifty_seconds;
	led_seconds[51] = fiftyone_seconds;
	led_seconds[52] = fiftytwo_seconds;
	led_seconds[53] = fiftythree_seconds;
	led_seconds[54] = fiftyfour_seconds;
	led_seconds[55] = fiftyfive_seconds;
	led_seconds[56] = fiftysix_seconds;
	led_seconds[57] = fiftyseven_seconds;
	led_seconds[58] = fiftyeight_seconds;

	led_minutes[0] = zero_minutes;
	led_minutes[1] = one_minutes;
	led_minutes[2] = two_minutes;
	led_minutes[3] = three_minutes;
	led_minutes[4] = four_minutes;
	led_minutes[5] = five_minutes;
	led_minutes[6] = six_minutes;
	led_minutes[7] = seven_minutes;
	led_minutes[8] = eight_minutes;
	led_minutes[9] = nine_minutes;
	led_minutes[10] = ten_minutes;
	led_minutes[11] = eleven_minutes;
	led_minutes[12] = twelve_minutes;
	led_minutes[13] = thirteen_minutes;
	led_minutes[14] = fourteen_minutes;
	led_minutes[15] = fifteen_minutes;
	led_minutes[16] = sixteen_minutes;
	led_minutes[17] = seventeen_minutes;
	led_minutes[18] = eighteen_minutes;
	led_minutes[19] = nineteen_minutes;
	led_minutes[20] = twenty_minutes;
	led_minutes[21] = twentyone_minutes;
	led_minutes[22] = twentytwo_minutes;
	led_minutes[23] = twentythree_minutes;
	led_minutes[24] = twentyfour_minutes;
	led_minutes[25] = twentyfive_minutes;
	led_minutes[26] = twentysix_minutes;
	led_minutes[27] = twentyseven_minutes;
	led_minutes[28] = twentyeight_minutes;
	led_minutes[29] = twentynine_minutes;
	led_minutes[30] = thirty_minutes;
	led_minutes[31] = thirtyone_minutes;
	led_minutes[32] = thirtytwo_minutes;
	led_minutes[33] = thirtythree_minutes;
	led_minutes[34] = thirtyfour_minutes;
	led_minutes[35] = thirtyfive_minutes;
	led_minutes[36] = thirtysix_minutes;
	led_minutes[37] = thirtyseven_minutes;
	led_minutes[38] = thirtyeight_minutes;
	led_minutes[39] = thirtynine_minutes;
	led_minutes[40] = fourty_minutes;
	led_minutes[41] = fourtyone_minutes;
	led_minutes[42] = fourtytwo_minutes;
	led_minutes[43] = fourtythree_minutes;
	led_minutes[44] = fourtyfour_minutes;
	led_minutes[45] = fourtyfive_minutes;
	led_minutes[46] = fourtysix_minutes;
	led_minutes[47] = fourtyseven_minutes;
	led_minutes[48] = fourtyeight_minutes;
	led_minutes[49] = fourtynine_minutes;
	led_minutes[50] = fifty_minutes;
	led_minutes[51] = fiftyone_minutes;
	led_minutes[52] = fiftytwo_minutes;
	led_minutes[53] = fiftythree_minutes;
	led_minutes[54] = fiftyfour_minutes;
	led_minutes[55] = fiftyfive_minutes;
	led_minutes[56] = fiftysix_minutes;
	led_minutes[57] = fiftyseven_minutes;
	led_minutes[58] = fiftyeight_minutes;

	led_hours[0] = one_hours;
	led_hours[1] = two_hours;
	led_hours[2] = three_hours;
	led_hours[3] = four_hours;
	led_hours[4] = five_hours;
	led_hours[5] = six_hours;
	led_hours[6] = seven_hours;
	led_hours[7] = eight_hours;
	led_hours[8] = nine_hours;
	led_hours[9] = ten_hours;
	led_hours[10] = eleven_hours;
	led_hours[11] = twelve_hours;
	led_hours[12] = thirteen_hours;
	led_hours[13] = fourteen_hours;
	led_hours[14] = fifteen_hours;
	led_hours[15] = sixteen_hours;
	led_hours[16] = seventeen_hours;
	led_hours[17] = eighteen_hours;
	led_hours[18] = nineteen_hours;
	led_hours[19] = twenty_hours;
	led_hours[20] = twentyone_hours;
	led_hours[21] = twentytwo_hours;
	led_hours[22] = twentythree_hours;
	led_hours[23] = twentyfour_hours;

}

void reset_leds()
{
	P1OUT = 0x08; // P1.3 high, all else low
	P2OUT = 0xB4; // P2.2, P2.4, P2.5, P2.7 high, all else low
	P3OUT = 0x9B; // P3.1, P3.2, P3.3, P3.4, P3.7 high, all else low
	P4OUT = BIT1; // P4.1 high
}

// LED pin "high" functions

// SECONDS
void zero_seconds()
{
	//Row 1, Column 1
	P1OUT = (P1_OFF ^ BIT3);
	P4OUT = (P4_OFF | BIT0);

	return;
}
void one_seconds()
{
	//Row 9, Column 12
	//P2OUT ^= BIT2;
	//P2OUT |= BIT0;
	P2OUT = (P2_OFF ^ BIT2) | BIT0;
	return;
}
void two_seconds()
{
	//Row 9, Column 11
	//P2OUT ^= BIT2;
	//P2OUT |= BIT1;
	P2OUT = (P2_OFF ^ BIT2) | BIT1;
	return;
}
void three_seconds()
{
	//Row 9, Column 10
	//P2OUT ^= BIT2;
	//P3OUT |= BIT5;
	P2OUT = (P2_OFF ^ BIT2);
	P3OUT = (P3_OFF | BIT5);
	return;
}
...
...
...
```

After some calculations, I determined I could drastically reduce the memory footprint required for the lookup table by stored the PxOUT assignments directly in a lookup table instead of functions that then point to the correct instructions:

```c
/*
 * led_controller.c
 *
 *  Created on: Aug 21, 2015
 *      Author: Sean
 */

#include <msp430.h>
#include "Headers\led_controller.h"


typedef struct {
    char P1;
    char P2;
    char P3;
    char P4;
    char PJ;
} ledCommand_t;

static char test = 0;
{% raw %}
static const ledCommand_t led_seconds[60] = {{ (char)P1_OFF,            (char)(P2_OFF ^ BIT3),              (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT0) },    // 0 seconds
                                             { (char)P1_OFF,            (char)((P2_OFF ^ BIT3) | BIT7),     (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },             // 1 second
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT3),              (char)(P3_OFF | BIT7),              (char)P4_OFF,           (char)PJ_OFF },             // 2 seconds
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT3) | BIT6),     (char)P4_OFF,           (char)PJ_OFF },                   // 3 seconds
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT3) | BIT4),     (char)P4_OFF,           (char)PJ_OFF },                   // 4 seconds
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT6),              (char)(P3_OFF ^ BIT3),              (char)P4_OFF,           (char)PJ_OFF },                   // 5 seconds
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT3),              (char)(P4_OFF | BIT0),  (char)PJ_OFF },                   // 6 seconds
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT3),              (char)P4_OFF,           (char)(PJ_OFF | BIT1) },    // 7 seconds
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT1),              (char)(P3_OFF ^ BIT3),              (char)P4_OFF,           (char)PJ_OFF },                   // 8 seconds
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT0),              (char)(P3_OFF ^ BIT3),              (char)P4_OFF,           (char)PJ_OFF },                   // 9 seconds
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT3),              (char)P4_OFF,           (char)(PJ_OFF | BIT3) },    // 10 seconds
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT3),              (char)P4_OFF,           (char)(PJ_OFF | BIT2) },    // 11 seconds
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT3),              (char)P4_OFF,           (char)(PJ_OFF | BIT0) },    // 12 seconds
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT7),              (char)(P3_OFF ^ BIT3),              (char)P4_OFF,           (char)PJ_OFF },                   // 13 seconds
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT3) | BIT7),     (char)P4_OFF,           (char)PJ_OFF },                   // 14 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)P2_OFF,                       (char)(P3_OFF | BIT6),              (char)P4_OFF,           (char)PJ_OFF },                   // 15 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)P2_OFF,                       (char)(P3_OFF | BIT4),              (char)P4_OFF,           (char)PJ_OFF },              // 16 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)(P2_OFF | BIT6),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },              // 17 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)P2_OFF,                       (char)P3_OFF,                       (char)(P4_OFF | BIT0),  (char)PJ_OFF },              // 18 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT1) },              // 19 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)(P2_OFF | BIT1),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },              // 20 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)(P2_OFF | BIT0),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },              // 21 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT3) },              // 22 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT2) },              // 23 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT0) },              // 24 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)(P2_OFF | BIT7),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },              // 25 seconds
                                             { (char)(P1_OFF ^ BIT4),   (char)P2_OFF,                       (char)(P3_OFF | BIT7),              (char)P4_OFF,           (char)PJ_OFF },              // 26 seconds
                                             { (char)(P1_OFF ^ BIT2),   (char)P2_OFF,                       (char)(P3_OFF | BIT6),              (char)P4_OFF,           (char)PJ_OFF },              // 27 seconds
                                             { (char)(P1_OFF ^ BIT2),   (char)P2_OFF,                       (char)(P3_OFF | BIT4),              (char)P4_OFF,           (char)PJ_OFF },              // 28 seconds
                                             { (char)(P1_OFF ^ BIT2),   (char)(P2_OFF | BIT6),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },              // 29 seconds
                                             { (char)(P1_OFF ^ BIT2),   (char)P2_OFF,                       (char)P3_OFF,                       (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT2),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                             { (char)(P1_OFF ^ BIT2),   (char)(P2_OFF | BIT1),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT2),   (char)(P2_OFF | BIT0),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT2),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                             { (char)(P1_OFF ^ BIT2),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT2) },
                                             { (char)(P1_OFF ^ BIT2),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                             { (char)(P1_OFF ^ BIT2),   (char)(P2_OFF | BIT7),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT2),   (char)P2_OFF,                       (char)(P3_OFF | BIT7),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT2) | BIT6),       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT2) | BIT4),       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT6),              (char)(P3_OFF ^ BIT2),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT2),              (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT2),              (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT1),              (char)(P3_OFF ^ BIT2),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT0),              (char)(P3_OFF ^ BIT2),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT2),              (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT2),              (char)P4_OFF,           (char)(PJ_OFF | BIT2) },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT2),              (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT7),              (char)(P3_OFF ^ BIT2),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT2) | BIT7),     (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT3),              (char)(P3_OFF | BIT6),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT3),              (char)(P3_OFF | BIT4),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)((P2_OFF ^ BIT3) | BIT6),     (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT3),              (char)P3_OFF,                       (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT3),              (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                             { (char)P1_OFF,            (char)((P2_OFF ^ BIT3) | BIT1),     (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)((P2_OFF ^ BIT3) | BIT0),     (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT3),              (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT3),              (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT2) }
};

static const ledCommand_t led_minutes[60] = {{ (char)P1_OFF,            (char)(P2_OFF ^ BIT4),              (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                             { (char)P1_OFF,            (char)((P2_OFF ^ BIT4) | BIT7),     (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT4),              (char)(P3_OFF | BIT7),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT0) | BIT6),     (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT0) | BIT4),     (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT6),              (char)(P3_OFF ^ BIT0),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT0),              (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT0),              (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT1),              (char)(P3_OFF ^ BIT0),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT0),              (char)(P3_OFF ^ BIT0),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT0),              (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT0),              (char)P4_OFF,           (char)(PJ_OFF | BIT2) },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT0),              (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT7),              (char)(P3_OFF ^ BIT0),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT0) | BIT7),     (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT5),   (char)P2_OFF,                       (char)(P3_OFF | BIT6),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT5),   (char)P2_OFF,                       (char)(P3_OFF | BIT4),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT5),   (char)(P2_OFF | BIT6),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT5),   (char)P2_OFF,                       (char)P3_OFF,                       (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT5),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                             { (char)(P1_OFF ^ BIT5),   (char)(P2_OFF | BIT1),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT5),   (char)(P2_OFF | BIT0),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT5),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                             { (char)(P1_OFF ^ BIT5),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT2) },
                                             { (char)(P1_OFF ^ BIT5),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                             { (char)(P1_OFF ^ BIT5),   (char)(P2_OFF | BIT7),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT5),   (char)P2_OFF,                       (char)(P3_OFF | BIT7),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT3),   (char)P2_OFF,                       (char)(P3_OFF | BIT6),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT3),   (char)P2_OFF,                       (char)(P3_OFF | BIT4),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT3),   (char)(P2_OFF | BIT6),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT3),   (char)P2_OFF,                       (char)P3_OFF,                       (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT3),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                             { (char)(P1_OFF ^ BIT3),   (char)(P2_OFF | BIT1),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT3),   (char)(P2_OFF | BIT0),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT3),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                             { (char)(P1_OFF ^ BIT3),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT2) },
                                             { (char)(P1_OFF ^ BIT3),   (char)P2_OFF,                       (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                             { (char)(P1_OFF ^ BIT3),   (char)(P2_OFF | BIT7),              (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)(P1_OFF ^ BIT3),   (char)P2_OFF,                       (char)(P3_OFF | BIT7),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT1) | BIT6),     (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT1) | BIT4),     (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT6),              (char)(P3_OFF ^ BIT1),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT1),              (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT1),              (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT1),              (char)(P3_OFF ^ BIT1),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT0),              (char)(P3_OFF ^ BIT1),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT1),              (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT1),              (char)P4_OFF,           (char)(PJ_OFF | BIT2) },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)(P3_OFF ^ BIT1),              (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                             { (char)P1_OFF,            (char)(P2_OFF | BIT7),              (char)(P3_OFF ^ BIT1),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)P2_OFF,                       (char)((P3_OFF ^ BIT1) | BIT7),     (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT4),              (char)(P3_OFF | BIT6),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT4),              (char)(P3_OFF | BIT4),              (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)((P2_OFF ^ BIT4) | BIT6),     (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT4),              (char)P3_OFF,                       (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT4),              (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                             { (char)P1_OFF,            (char)((P2_OFF ^ BIT4) | BIT1),     (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)((P2_OFF ^ BIT4) | BIT0),     (char)P3_OFF,                       (char)P4_OFF,           (char)PJ_OFF },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT4),              (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                             { (char)P1_OFF,            (char)(P2_OFF ^ BIT4),              (char)P3_OFF,                       (char)P4_OFF,           (char)(PJ_OFF | BIT2) }

};

static const ledCommand_t led_hours[24] = {{ (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                           { (char)(P1_OFF ^ BIT1),     (char)(P2_OFF | BIT7),  (char)P3_OFF,           (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)(P3_OFF | BIT7),  (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)(P3_OFF | BIT6),  (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)(P3_OFF | BIT4),  (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)(P2_OFF | BIT6),  (char)P3_OFF,           (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                           { (char)(P1_OFF ^ BIT1),     (char)(P2_OFF | BIT1),  (char)P3_OFF,           (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)(P2_OFF | BIT0),  (char)P3_OFF,           (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT2) },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT0) },
                                           { (char)(P1_OFF ^ BIT1),     (char)(P2_OFF | BIT7),  (char)P3_OFF,           (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)(P3_OFF | BIT7),  (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)(P3_OFF | BIT6),  (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)(P3_OFF | BIT4),  (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)(P2_OFF | BIT6),  (char)P3_OFF,           (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)(P4_OFF | BIT0),  (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT1) },
                                           { (char)(P1_OFF ^ BIT1),     (char)(P2_OFF | BIT1),  (char)P3_OFF,           (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)(P2_OFF | BIT0),  (char)P3_OFF,           (char)P4_OFF,           (char)PJ_OFF },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT3) },
                                           { (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT2) }
                                           //{ (char)(P1_OFF ^ BIT1),     (char)P2_OFF,           (char)P3_OFF,           (char)P4_OFF,           (char)(PJ_OFF | BIT0) }
};

void reset_leds()
{
    P1OUT = (char)P1_OFF;
    P2OUT = (char)P2_OFF;
    P3OUT = (char)P3_OFF;
    P4OUT = (char)P4_OFF;
    PJOUT = (short)PJ_OFF;
}


#pragma FUNC_ALWAYS_INLINE(LED_SetCurrentLED)
inline void LED_SetCurrentLED(unsigned char index, selectedRow_t row)
{
    reset_leds();
    switch(row)
    {
    case HOURS_ROW:
    {
            P1OUT = led_hours[index].P1;
            P2OUT = led_hours[index].P2;
            P3OUT = led_hours[index].P3;
            P4OUT = led_hours[index].P4;
            PJOUT = (PJOUT & 0xFF) | (short) led_hours[index].PJ;

        break;
    }
    case MINUTES_ROW:
    {
            P1OUT = led_minutes[index].P1;
            P2OUT = led_minutes[index].P2;
            P3OUT = led_minutes[index].P3;
            P4OUT = led_minutes[index].P4;
            PJOUT = (PJOUT & 0xFF00) | (short) led_minutes[index].PJ;

        break;
    }
    case SECONDS_ROW:
    {
            P1OUT = led_seconds[index].P1;
            P2OUT = led_seconds[index].P2;
            P3OUT = led_seconds[index].P3;
            P4OUT = led_seconds[index].P4;
            PJOUT = (PJOUT & 0xFF00) | (short) led_seconds[index].PJ;
       break;
    }
    default:

        break;
    }
}
{% endraw %}
```

With the additional memory now avaiable, I was free to program in different display modes for how the LED show times:
* Minimal - 1 LED shown for hour, 1 LED shown for minute, 1 LED shown for seconds
* Minimal Extended - Line of LEDs starting from "seconds" row to minute or hour row. Example: 2:34.15 would light up LED 10, 34, and 15 in the "seconds" row, LED 10 and 34 in the "minutes" row, and LED 2 in the "hours" row.
* Revolving - All LEDs in the cooresponding row up to the current second, minute, hour are lit.

These three modes and the other logic to read button input and set time took up most of the 4 KB of available memory with some compiler optimizations enabled. But I did have a functioning watch at this point. 

I also created a small portion of small code to test the ambient light functionality. The light detection works by using the LEDs as a basic photo diode when they are in reverse-bias mode. My post on [LED Light Detection](/led-light-detection) explains this method of light detection in more detail. The software demo detects the amount of light currently hitting the hour LEDs and changes lights up a certain LED in the "seconds" row depending on the light level measured.

![LED light sense demo](/assets/images/light-sensing-demo.gif)

### LED Watch Enclosure
As I was creating the hardware design revisions for LED Watch v1.1, I started thinking about how I could potentially package the PCB in a way that could be worn. My goal was to make this look like a typical and classic watch outside of the actual watch face. I considered 3D printing a lot as it's rather inexpensive to print out objects in 2018. I however felt it would be a huge effort to design a 3D model of an enclosure and I had no idea on how to incorporate a piece of glass or plastic to protect the internal PCB. The next option I had in mind was finding an enclosure that was already out in the market and available for purchasing. 

I expected there would be a huge market for replacement watch parts but I was having trouble finding cases standalone. Furthermore, they didn't have buttons in the same location as my PCB. As I searched more and more, it appeared 3D printing would be my best option until I stumbled upon a LED touchscreen watch that was being sold from China for very cheap. After looking at the size specifications, I was surprised to find the watch face was about the same size as my PCB - some really good luck there I must say. I decided to order one and rip out the internals to see if my PCB would actually fit inside. Once it arrived, I confirmed my PCB fit fairly snug after some slight sanding of the PCB edges. The watch bands were definitely the cheapest and most uncomfortable I had ever seen but replaceable thankfully.

However, there was one issue with the watch case - it didn't have any buttons as it was originally intended to be touch sensor. I had two ideas: use the light sensing capability as user input instead of the buttons or drill holes into the watch case and 3D print buttons. I created a list of pros and cons for each to help make up my mind.

__Light sensing user input:__

_Pros_
* No need to modify the existing case. More water-proof
* A bit of a cool factor you don't get with regular buttons

_Cons_
* No way to interact with the watch in complete darkness. If you want to see the time in the middle of the night with no light, you're out of luck
* More complex software needed to differentiate between different types of user input (Ex: if user covers the LEDs for 2 seconds, do task A. If user covers for 4 seconds, do task B)
* Accidental toggling of display by walking into another area with very large difference in ambient light.

__3D printed buttons:__

_Pros_
* Less likely chance of accidental user input
* Buttons allow for more user input combinations
* Much simpler software

_Cons_
* Need to drill holes into the case and design a button that fits in the hole without falling out of case unintentionally.
* Much less water-proof

A work colleague of mine interested in the project offered to test print some buttons with his 3D printer and had a better work bench setup that allowed him to drill much more precise holes into the case than I could with my tools. Since Gen 2 was still a prototype, I decided to modify my test watch case to see if 3D printing buttons was a viable option. With his help, a case with working buttons was made.

![Metal watch case with drilled holes](/assets/images/drilled-case.jpg "Watch case with drilled holes")

Although I didn't like how the case was no longer waterproof with the drilled holes, it did provide better functionality when it came to user-input. If the receivers of the watches preferred the light detection method instead, I could provide the source code and they could purchase a new case without the drill holes. If I went with the light-detection as default, they would then have to drill the holes and 3D print the buttons themselves. This isn't as easy in my opinion.

![Metal watch with PCB inside](/assets/images/led-watch-v11-in-case.jpg "Gen 2 inside watch case")

There was actually another issue with the case and PCB design version 1.1 - I had to sand down the edge extension off the bottom of the PCB used to connect a 0.05" pin header. The issue wasn't sanding but instead the exposed vias on the edge of the board and how they contacted the metal watch case. The four pins on that pin header are VCC, GND, TDO, and TCLK. If the VCC and GND pin both touch the case, the battery will become shorted out. For version 1.1 this is resolved by solving the exposed vias with kapton tape before inserting into the case. This can easily be fixed in the next revision by rotating the extension by 20&#176 left or right.

![Image of issues]

For a detailed review on version 1.1, see <https://seanmharrington.github.io/led-watch-v11/>.

## Version 1.11 - The final version?
Although I really appreciate OSHPark's service and their purple PCBs, it isn't exactly what I want the final product to look like. Plus, I had a few issues from Gen 2 that needed fixing in the board layout. It didn't take long before I had a new board revision and I was ready to print some more boards. I decided I would make 10 watches and hopefully finish up this project for good. Looking around for other PCB manufacturers, I found <https://www.shenzhen2u.com/PCB> which provides PCB prototyping with fairly impressive allowed tolerances as a very competitive price. They also offer professional PCB manufacturing that offers much more options and improved tolerances but I elected to go with the prototyping service as they had the options I needed and I was on a budget. I decided to go with white for the soldermask this time as I felt it would cause the LEDs to not pop out so much when they aren't on.

![Image of Version 1.11 PCB]

I'll be the first to say that the finish on these PCBs wasn't the greatest. In many places where components are close together, soldermask is missing between pads. Perhaps this was an issue with the OSHPark boards as well but it's harder to notice due to the dark purple soldermask compared to the white soldermask on these boards. Even though the soldermask isn't perfect, the boards still perform as intended. I assembled a board and flashed the watch software and confirmed the hardware issues I had before were now gone.

![LED Watch v1.11 Dark](/assets/images/led-watch-banner-1.1.jpg "LED Watch v1.11 in the dark")

For a detailed review on version 1.11, see <https://seanmharrington.github.io/led-watch-v111/>.

## Conclusion
To be determined... Time to finish enough of these watches for the wedding!

## LED Watch Blog Posts
All blog posts related to this project can be found below.

{% for post in site.posts %} 
	{% if post.tags contains 'Wristwatch' %}
		
	{% endif %}
{% endfor %} 
