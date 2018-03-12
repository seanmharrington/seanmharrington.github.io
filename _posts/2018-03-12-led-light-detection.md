---
title: Using LEDs to Detect Light
comments: true
classes: wide
header:
  teaser: /assets/images/light-sensing-demo.gif
---

About a year ago, I came upon an article about using LEDs as an alternative to typical photodiode sensors. This was the first time I had seen this capability with LEDs and it seemed like it would be perfect to test out and implement in my LED wristwatch design.

Sparkfun wrote a short post on using LEDs as photodiodes found here: <https://www.sparkfun.com/news/2161>

I will attempt to summarize the science behind using LEDs as photodiodes and then show a working example of using LEDs to detect light with demo software I wrote for my LED Watch.

## The Science

<figure>
	<a href="/assets/images/PN_diode_with_electrical_symbol.svg"><img src="/assets/images/PN_diode_with_electrical_symbol.svg" style="background-color:white"></a>
	<figcaption> By Raffamaiden - Own work, CC BY-SA 3.0, https://commons.wikimedia.org/w/index.php?curid=21285768 </figcaption>
</figure>

Light emitting diodes are basically simple diodes with special doping materials that cause light to be emitted when current flows through them. Diodes can be powered in two different way: forward bias and reverse bias. Forward bias means applying a positive voltage on the P-region of a diode (P-type silicon in the image above) and reverse bias means the positive voltage is placed on the N-region side (N-type silicon also known as cathode).

In a forward bias configuration, the holes in the P-region and electrons in the N-region of the diode are pushed together. With enough voltage, the holes and electrons continue to get closer together until the energy from the increasing voltage overcomes the repelling forces between the doped zones and electrons/holes. This causes current to finally flow across the diode and in the case of an LED, light is emitted.

<figure class="half">
	<a href="/assets/images/Diode-IV-Curve.svg"><img src="/assets/images/Diode-IV-Curve.svg" style="background-color:white"></a>
	<a href="/assets/images/PnJunction-LED-E.svg"><img src="/assets/images/PnJunction-LED-E.svg" style="background-color:white"></a>
	<figcaption>
		<p>
		Figure 1 - Diode Current, Voltage Curve. By H1voltage - Roughly derived from http://en.wikipedia.org/wiki/File:Rectifier_vi_curve.GIF. Vectorization is own work., CC BY-SA 3.0, https://commons.wikimedia.org/w/index.php?curid=5724764
		<br>
		Figure 2 - LED Forward Bias - By User:S-kei - File:PnJunction-LED-E.PNG, CC BY-SA 2.5, https://commons.wikimedia.org/w/index.php?curid=14985902
		</p>
	</figcaption>
</figure>

In forward-bias mode, Vd is the amount of voltage required for light to start emitting. 

As seen from Figure 1, there is also a reverse bias zone and breakdown zone for diodes. The LED starts to act like a photodiode when in reverse-bias mode which is what we are really interested in exploring here. 

To understand the behavior, we can first look at the reverse-bias behavior of a diode. When a positive voltage is applied to the N-region of a diode, the electrons and holes separate further increasing the depletion region (zone where no charge carriers exist due to an electric field). This in turn causes the LED to act like a capacitor. If the voltage is then removed from the LED, the voltage across the LED will discharge over time until it's 0 volts. 

Repeating the process by placing the LED in reverse-bias mode, removing the voltage source, and then applying more light to the LED will cause electron-hole pairs to form. These will move and cancel out with existing electrons and holes already present. This is turn causes the voltage across the LED to drain more quickly.

Therefore, the rate of voltage discharge will vary based on the amount of light present after removing a voltage source across a LED in a reverse-bias configuration. If we can measure the small difference in discharge times, we can determine when there is a change in the amount of light present. To do this easily, a microcontroller that has a built-in ADC can be used.

## Circuit Configuration

I'm going to focus on a circuit that can both light up a LED and detect light at the same time. As I mentioned above, to light up an LED it must have a forward-bias voltage applied to it. In order to measure light however, we need a reverse-bias voltage. For that reason, we can't physically have both the LED on and measuring light at the same time. We however can trick the eye into believing that both are actually happening simultaneously... To do this, we need to perform the light sensing routine at a speed our eyes can't detect.

To get started, let's look at the datasheet for the microcontroller I'm using in my LED wristwatch project: MSP430FR5721

![MSP430FR5721 DA Package](/assets/images/msp430fr5721_pinout.jpg "MSP430FR5721 DA Package Pinout")

To detect light with 1 LED, we will need 2 different pins - 1 ADC and 1 GPIO. In the figure above, GPIOs are pins that start with "P" following by two numbers separated by a decimal such as P1.2 on pin 7. Available ADCs are pins that have "A" followed by 2 numbers such as A15 on pin 11.

Each pin can be reconfigured for a different purpose within software which is why each pin has many different labels. It's important to note that we will need to be able to reconfigure our ADC as a GPIO. This is possible for every pin that has ADC capabilities on this microcontroller.

We will therefore wire up an LED in series with a resistor between pins 7 and 11. It's important that the LED anode is connected to the pin that can be programmed as an ADC as shown below:

![Image]

In my case, I used a 200 Ohm current-limiting resistor as it maximized the continuous current draw of my LED (the MSP430 runs at 3.3 volts). This resistor value will vary based on the voltage output of the microcontroller and LED you are using.

### Turning on the LED

To light up the LED in this circuit, we will program both pin 7 and 11 to be a GPIO. The LED will light up when we place a forward-bias voltage across the LED. To do so, we command pin 11 to output high (3.3V) and command pin 7 to be an output in a low state (GND). To turn off the LED, pin 11 can be commanded to output low like pin 7.

### Detecting light with LED

The following is a quick outline of the steps required to use the LED as a photodiode:
1. Generate a reverse-bias voltage across the LED by changing pin 11 to output low and pin 7 to output high.
2. Change pin 11 to be in ADC mode
3. Immediately measure the voltage at the ADC
4. Wait a certain amount of fixed time (depends on LED being used and how fast it discharges)
5. Measure the voltage again at the ADC
6. Take the difference between the first measurement and second measurement
7. The amount of light measured can be determined by how large or small the voltage difference is
8. Change pin 11 back to GPIO
9. Change pin 11 to output high and pin 7 to output low to briefly turn on LED and drain remaining charge
10. Return to step 1 and repeat

If we take the voltage difference measured between the two points in time and divide by the measurement time, we get the average voltage for that measurement. This is the area under the curve or integral. If the process is repeated again, the two averages can be compared to determine how much the light changed between the two measurements.

![Image of integral of voltage over time]

The voltage difference measured between each cycle of steps will most likely vary slightly even with the same amount of light. A averaging filter can help reduce the influence of these small voltage differences. 

### Detecting light with LED matrix
In my LED wristwatch project, I'm using a 12x11 LED matrix to display the time. It's possible to also add light-sensing capabilities to an LED matrix by expanding onto the circuit we originally used for a single LED. Simply connect the shared anode of each array to an ADC on the microcontroller.

![Image of 12x11 matrix]

## Demo Software

The following code uses the light detected by 1 LED in the matrix to determine which LED representing the seconds of time should be turned on.

```c
#include <msp430.h>
#include "Headers/head.h"
#include "Headers/led_controller.h"
//#include <ctpl.h>

/*
 * main.c
 */

volatile unsigned int ADC_Result = 0;

void main(void)
{
    WDTCTL = WDTPW + WDTHOLD;    // Stop watchdog timer

    initialize();

    ADC10CTL0 |= ADC10SHT_0 + ADC10ON;
    ADC10CTL1 |= ADC10SHP;
    ADC10CTL2 |= ADC10RES;
    ADC10MCTL0 |= ADC10INCH_1;
    ADC10IE |= ADC10IE0;


    unsigned int startV = 0;
    unsigned int endV = 0;
    while(1)
    {
        reset_leds();

        LED_SetCurrentLED(0, 0);        // Turn on 12th hour LED

        reset_leds();                   // Turn off 12th hour LED

        P1OUT = (char)(P1_OFF | BIT1);   // Charge LED up by reversing current through it
        PJOUT = (char)(PJ_OFF ^ BIT0);
        P2OUT = (char)(P2_OFF ^ (BIT0 + BIT1 + BIT6 + BIT7));
        P3OUT = (char)(P3_OFF ^ (BIT4 + BIT6 + BIT7));
        P4OUT = (char)(P4_OFF ^ BIT0);

        reset_leds();                   // Turn off 12th hour LED

        P1DIR = 0x3D;

        // Change P1.1 to be ADC input
        P1SEL0 |= BIT1;
        P1SEL1 |= BIT1;

        // Trigger ADC sample start
        ADC10CTL0 |= ADC10ENC + ADC10SC;

        // Sleep until ADC done sampling and result is stored
        __bis_SR_register(CPUOFF + GIE);
        __no_operation();

        // Store current voltage before discharge
        startV = ADC_Result;

        __delay_cycles(500000);

        // Measure voltage after discharge time
        // Trigger ADC sample start
        ADC10CTL0 |= ADC10ENC + ADC10SC;

        // Sleep until ADC done sampling and result is stored
        __bis_SR_register(CPUOFF + GIE);
        __no_operation();

        endV = ADC_Result;

        unsigned int diff = 0;


        if (startV > endV)
        {
            diff = startV - endV;
        }

        // 3.3v / 60 = Every 0.055 V is a LED
        unsigned int LED = 1;
        while (diff > 19)
        {
            diff -= 18;
            LED ++;
        }

        if (LED > 59)
        {
            LED = 59;
        }

        P1DIR = 0x3F;
        P1SEL0 &= ~BIT1;                // Change P1.1 to be digital output again
        P1SEL1 &= ~BIT1;

        LED_SetCurrentLED(LED, 1);      // Turn on minute representing amount of light detected

        __delay_cycles(500);            // Hold that LED
    }

}

void initialize()
{
	//GPIO Setup
	//
	// Data register setup for PxDIR, PxOUT
	//   | Px.7 | Px.6 | Px.5 | Px.4 | Px.3 | Px.2 | Px.1 | Px.0 |
	//

	//P1SEL = 0xEF;
	P1DIR = 0x3F;       // All P1 ports outputs except P1.6, P1.7
	P1OUT = 0x3E;       // All outputs high except P1.0. P1.6, P1.7 pulldown selected
	P1REN |= BIT6;      // Enable pulldown for P1.6
	P1IES &= ~(BIT6);   // Interrupt on low -> high
	P1IE |= BIT6;       // Enable interrupt for B1

	P2DIR = 0xDB;       // All P2 ports outputs except P2.3, P2.5
	P2OUT = 0x18;       // All outputs low except P2.3, P2.4. P2.2, P2.5 pulldown selected
	P2REN |= BIT2;      // Enable pulldown for P2.2
	P2IES &= ~(BIT2);   // Interrupt on low -> high
	P2IE |= BIT2;       // Enable interrupt for B2

	P3DIR = 0xDF;   // All P3 ports output except P3.5
	P3OUT = 0x0F;   // All outputs low, P3.5 pulldown selected

	P4DIR = BIT0;   // P4.0 is a output;
	P4OUT = 0x00;   // All outputs low, P4.0 pulldown selected

	PJDIR = 0x0F;   // PJ.0 - PJ.3 set as outputs
	PJOUT &= ~(BIT0 | BIT1 | BIT2 | BIT3);  // PJ.0 - PJ.3 outputs low

	// For XT1
	// External clock (XT1) and system clocks setup
	PJSEL0 |= BIT4 + BIT5;


	CSCTL0_H = 0xA5;						// Unlock clock register
	//CSCTL1 |= DCOFSEL0 + DCOFSEL1;             // Set max. DCO setting (8)
	CSCTL1 |= DCOFSEL_3;
	CSCTL2 = SELA_0 + SELS_3 + SELM_3; 		//ACLK = XT1 and MCLK = DCO
	//CHANGE CSCTL3 DIVIDERS!
	//CSCTL3 = DIVA_0 + DIVS_1 + DIVM_1;
	CSCTL3 = DIVA_0 + DIVS_5 + DIVM_0;      // ACLK / 1, SMCLK / 32 (250 kHz), MCLK / 1 (8 MHz)
	CSCTL4 = XT1DRIVE_0;
	CSCTL4 &= ~XT1OFF;

	do
	{
		CSCTL5 &= ~XT1OFFG;		// Clear XT1 fault flag
		SFRIFG1 &= ~OFIFG;
	} while (SFRIFG1&OFIFG);
	CSCTL0_H = 0x01; // Lock clock register


	// Turn off temp sensor
	REFCTL0 |= REFTCOFF;
	REFCTL0 &= ~OFIFG;

	__bis_SR_register(GIE); //Enable interrupts
}


#pragma vector=ADC10_VECTOR
__interrupt void ADC10_ISR(void)
{
    switch(__even_in_range(ADC10IV,12))
    {
    case 12:
    {
        ADC_Result = ADC10MEM0;
        __bic_SR_register_on_exit(CPUOFF);
        break;
    }
    default:
        break;
    }
}
```
led_controller.c: <https://github.com/seanmharrington/LED_Watch/blob/v1.1/Software/LED_Watch/led_controller.c>

Here's a video of the demo functioning:

![](/assets/images/light-sensing-demo.gif "LED light sensing demo")

