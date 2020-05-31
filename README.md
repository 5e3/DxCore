[![Join the chat at https://gitter.im/SpenceKonde/megaTinyCore](https://badges.gitter.im/SpenceKonde/megaTinyCore.svg)](https://gitter.im/SpenceKonde/megaTinyCore?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)


### [Wiring](Wiring.md)
### [Installation](Installation.md)
### [Making a cheap UPDI programmer](MakeUPDIProgrammer.md)

# DACore - Arduino support for DA-series parts
This is a very early version of the Arduino core to support the new DA-series AVR microcontrollers from Microchip. These are the latest and highest spec 8-bit AVR microcontrollers from Microchip - it is likely that these were what would have been the megaAVR 1-series parts, but these have taken the product line to a whole new level.  With up to 128k flash, 16k SRAM, 55 I/O pins, 6 UART ports, 2 SPI and I2C ports, and all the exciting features of the tinyAVR 1-series and megaAVR 0-series parts like the event system, the modern timers, the improved interrupts... but for almost every major system, they've added some significant improvement. You liked the type A timer? Well now you have two of them. You wished you could make a type B timer count events? You can do that. Combine two TCB's into one so you can time longer events? Yup. More bits of accuracy on the ADC? Yup - this is a 12-bit ADC - and the DAC is 10-bit too!

These parts depart from the naming scheme used for AVR devices in the past; these are named AVR followed by the size of the flash, in KB, followed by DA, then the number of pins. At this time, only the 128K parts are available; the AVR128DA datasheet has listed planned memory capacities for the available pin counts. Note that the pin count also determines how many of certain peripherals the parts have available - parts with more pins have more peripherals to do things with those pins.

This core was based on megaTinyCore; as documentation is updated, this will be changed.


## Supported Parts (click link for pinout diagram and details)
* [AVR128DA28, AVR64DA28, AVR32DA28](megaavr/extras/DA28.md)
* [AVR128DA32, AVR64DA32, AVR32DA32](megaavr/extras/DA32.md)
* [AVR128DA48, AVR64DA48, AVR32DA48](megaavr/extras/DA48.md)
* [AVR128DA64, AVR64DA64](megaavr/extras/DA64.md)

## Supported Clock Speeds
Subject to change - currently speed grades are not known (datasheet implies that all speeds are supported over the whole voltage range - this seems unlikely)
* 24MHz Internal
* 20MHz Internal
* 16MHz Internal
* 12MHz Internal
*  8MHz Internal
*  4MHz Internal
*  1MHz Internal
Additionally, experimentation has shown that it is possible to write reserved values to the FREQSEL bits of CLKCTRL.OSCHFCTRLA to overclock the chip. The core currently supports the following overclocked speeds - however, these will almost certainly not work across the full operating temperature or voltage range, and may behave erratically. Higher frequencies are not supported because in my tests, stuff stopped working at 36MHz and up. Use at own risk - and bear in mind that you probably don't need this!
* 28MHz Internal (overclocked)
* 32MHz Internal (overclocked)

There are multiple ways to generate some of the lower frequencies (do you prescale from higher frequency, or set the oscillator to the desired one? Suspect the latter is more pwoer efficient, but with the former you could still use the PLL - but are people going to want to?

These parts do not support using an external crystal like the classic AVR parts do, however the internal oscillator is tightly calibrated enough that the internal clock should work for UART communication, and an external watch crystal can be used to "auto-tune" the crystal frequency.

## The hardware reset pin is shared with UPDI programming.
The pin PA0 is the UPDI *or* reset pin depending on fuse settings (it could also be set as a GPIO pin, but we already have two very compelling uses for said pin)

This has one very troubling consequence - if we want to program via UPDI, we must not make that pin a Reset pin, otherwise we would need to use an HV programmer to reset the chip to allow UPDI programming again. At present this process is beyond the scope of this guide (apparently on some silicon revs, it is trivially easy to do, while on others it requires careful timing and a magic KEY signal). I hope to be able to point y'all to an affordable re-UPDIFIER in the near future.

There is a solution to the lack of a hardware reset pin when you need to keep UPDI programming, however - and that is that these parts support a software reset. Such a reset can be achieved with the simple command:
`_PROTECTED_WRITE(RSTCTRL.SWRR,1);`

If you were to set up a "low level" interrupt on a pin, and put that in the ISR, that pin would act as an ersatz reset pin (note that it wouldn't "stay" in reset like a real reset pin - it will reset the chip, and then the sketch (or bootloader) will start running again from the start until the interrupt was enabled, then reset again).

megaTinyCore supports configurations with this pin fused to act as reset, or as GPIO in Optiboot configurations. This is because it is still possible to program the board ove the UART serial using the bootloader with the UPDI functionality disabled. By default, when UPDI is set as reset, the standard boot-reset source behavior is used - it will enter the bootloader only when reset by a Software Reset, or by the Reset pin (allowing the usual DTR reset trick, but the sketch can start instantly on power-pn. When the pin is set as UPDI or GPIO, the optiboot bootloader used will also enter the bootloader on a power-on reset, and stay there for 8 seconds to faciiitate bootloading (and will also enter the bootloader on a software reset. Work is ongoing on a design for a low-cost arduino-based solution for 12v programming. As I am able to validate that and integrate support, we will introduce GPIO and reset for all configurations.

**Limited input facilities on PA0 with pin set as UPDI**
It has been discovered that as long as the patterns applied to the UPDI pin are not mistaken for a UPDI signal, the pin will still act as a fully functional input - for analogRead(), digitalRead(), and as input0 to CCL0. Obviously I cant's

# Features

### Memory-mapped flash: No need to declare PROGMEM or use F() macro anymore!
Unlike classic AVRs, on the megaavr parts, the flash is within the same address space as the main memory. This means pgm_read_*_near functions are not needed to read them. Because of this, the compiler automatically puts any variable declared `const` into progmem, and accesses it appropriately - you no longer need to explicitly declare them as PROGMEM. This includes quoted string literals, so the F() macro is no longer needed either (as of 1.0.6, the F() macro is a no-op).

However, do note that if you explicitly declare a variable PROGMEM, you must still use the pgm_read functions to read it, just like on classic AVRs - when a variable is declared PROGMEM on megaavr parts, the pointer is offset (address is relative to start of flash, not start of address space); this same offset is applied when using the pgm_read_*_near functions. Do note that declaring things PROGMEM and accessing with pgm_read_*_near functions, although it works fine, is slower and wastes a small amount of flash (compared to simply declaring the variables const).

**WARNING** In versions of megaTinyCore 1.0.6 and earlier, the sketch size reported during compilation does not include variables declared const. PROGMEM variables, however, are reported normally. in 1.1.0 and subsequent releases, the flash used by const variables is correctly reported.

#### Ways to refer to pins
The simple matter of how to refer to a pin for analogRead() and digitalRead(), particularly on non-standard hardware, has been a persistent source of confusion among Arduino users. It's my opinion that much of the blame rests with the decisions made by the Arduino team regarding how pins were to be referred to; the designation of some pins as "analog pins" leads people to think that those pins cannot be used for digital input, and the fact that all of the pins were renumbered fuirther muddies the water. The fact that many ATmega parts have a confusing mapping of ports to physical pins (which, thankfully, is no longer a problem on the megaavr parts), and the inconsistent decisions made by authors of subsequent hardwarte packages in an attempt to make those parts "more like an Arduino Uno", or (in some cases) to make the functions to map arduino pin numbers to PORT registers has led to further confusion around this topic.

This core uses a simple scheme for assigning the Arduino pin numbers: Pins are numbered starting from the the I/O pin closest to Vcc as pin 0 and proceeding counterclockwise, skipping the (mostly) non-usable UPDI pin. The UPDI pin is then assigned to the last pin number - as noted above, it is possible to read the voltage on the UPDI pin - we recommend this only as a last resort. On unofficial parts like these, we recommend that pins be referred to by the PIN_Pxn constants - this will maximize portability of your code and make it easier to look up information on the pins you are using in the relevant datasheets should that be necessary.

#### PIN_Pxn Port Pin Numbers (recommended)
**This is the recommended way to refer to pins** Defines are also provided of form PIN_Pxn, where x is A, B, or C, and n is a number 0~7 - (Not to be confused with the PIN_An defines described below). These just resolve to the digital pin number of the pin in question - they don't go through a different code path or anything. However, they have particular utility in writing code that works across the product line with peripherals that are linked to certain pins (by Port), as most peripherals are. Several pieces of demo code in the documentation take advantage of this.  Direct port manipulation is possible on the megaavr parts - and in fact several powerful additional options are available for it - see [direct port manipulation](megaavr/extras/DirectPortManipulation.md).


#### Arduino Pin Numbers
When a single number is used to refer to a pin - in the documentation, or in your code - it is always the "Arduino pin number". These are the pin numbers shown in orange (for pins capable of analogRead()) and blue (for pins that are not) on the pinout charts. All of the other ways of referring to pins are #defined to the corresponding Arduino pin number.
#### An and PIN_An constants
The core also provides An and PIN_An constants (where n is a number from 0 to 11). These refer to the ADC0 *channel* numbers. This naming system is similar to what was used on many classic AVR cores - on some of those, it is used to simplify the code behind analogRead() - but here, they are just #defined as the corresponding Arduino pin number. These are not shown on the pinout charts, as this is a deprecated way of referring to pins. The mapping of analog channels to pins is shown in the the datasheet under the I/O Multiplexing Considerations chapter. There are additionally PIN_An defines for compatibility with the official cores - these likewise point to the digital pin number associated with the analog channel. Note that channel A0 is on the UPDI/Reset pin - however, even when configured as UPDI, it can be used as an input as long as the signals it can be exposed to do not look like the UPDI enable sequence.

### Serial (UART) Support
All of these parts have a single hardware serial port (UART). It works exactly like the one on official Arduino boards (except that there is no auto-reset, unless you are using Optiboot and have configured that pin to act as reset, or have wired up an "ersatz reset pin" as described above). See the pinout charts for the location of the serial pins.

On all parts, the UART pins can be swapped to an alternate location.

On 2.0.0 and later, this is configured using the Wire.swap() or Wire.pins() methods. Both of them achieve the same thing, but differ in how you specify the set of pins to use. This should be called **before** calling SPI.begin().

`Serial.swap(1) or Serial.swap(0)` will set the the mapping to the alternate (1) or default (0) pins. It will return true if this is a valid option, and false if it is not (you don't need to check this, but it may be useful during development). If an invalid option is specified, it will be set to the default one.

`Serial.pins(TX pin, RX pin)` - this will set the mapping to whichever mapping has the specified pins as TX and RX. If this is not a valid mapping option, it will return false and set the mapping to the default. This uses more flash than Wire.swap(); that method is preferred.

In versions prior to 2.0.0, this was instead configured using the Tools -> UART Pins menu.

To maximize the accuracy of the baud rate, from the Tools -> Voltage for UART Baud menu, select whether the voltage is closer to 5v or 3v - the factory calibration supplies an oscillator error adjustment for the purpose of UART baud calculation for 5v and 3v, and using the right one will produce a baud rate closer to the target value. That said, testing has indicated that either setting is almost always good enough.

Bootloaders are available for both UART mappings; the UART bootloader option (prior to 2.0.0, the UART Pins option) selected when you do "burn bootloader" for an Optiboot board definition is the serial port that the uploaded bootloaded will use. You may freely change this when compiling/uploading sketches to use the other pins - the pins used by the bootloader will only change when you do "burn bootloader". **If this is your first time bootloading the board in question, and you want to turn UPDI into a Reset pin, burn bootloader first with the UPDI pin left as UPDI, so you can verify that, with the desired UART option, the bootloader really does try to use the pins you want it to - before you turn UPDI into reset and render the part unprogrammable.**

When operating at 1MHz, the UART can output 56700 baud, but not 115200 baud.

### SPI support
All of these parts have a single hardware SPI peripheral. It works exactly like the one on official Arduino boards using the SPI.h library. See the pinout charts for the location of these pins. Note that the 8-pin parts (412, 212, 402, 204) do not have a specific SS pin.

On all parts except the 14-pin parts, the SPI pins can be moved to an alternate location (note: On 8-pin parts, the SCK pin cannot be moved).

On 2.0.0 and later, this is configured using the Wire.swap() or Wire.pins() methods. Both of them achieve the same thing, but differ in how you specify the set of pins to use. This should be called **before** calling SPI.begin().

`SPI.swap(1) or SPI.swap(0)` will set the the mapping to the alternate (1) or default (0) pins. It will return true if this is a valid option, and false if it is not (you don't need to check this, but it may be useful during development). If an invalid option is specified, it will be set to the default one.

`SPI.pins(MOSI pin, MISO pin, SCK pin, SS pin);` - this will set the mapping to whichever mapping has the specified pins. If this is not a valid mapping option, it will return false and set the mapping to the default. This uses more flash than SPI.swap(); that method is preferred.

In versions prior to 2.0.0, this was instead configured using the Tools -> SPI Pins submenu; this is set at compile time (reburning bootloader is not required). In these versions, you could see whether the alternate pins are in use by checking if SPIREMAP is #defined - you can for example use it to check that the correct options are selected and terminate compilation so you can select the right option if that is the case.

This core disables the SS pin when running in SPI master mode. This means that the "SS" pin can be used for whatever purpose you want - unlike classic AVRs, where this could not be disabled. Earlier versions of this document incorrectly stated that this behavior was enabled in megaTinyCore; it never was, and SS was always disabled. It should be reenabled and the SS pin configured appropriately (probably as INPUT_PULLUP) if master/slave functionality is required.

### I2C (TWI) support
All of these parts have a single hardware I2C (TWI) peripheral. It works exactly like the one on official Arduino boards using the Wire.h library. See the pinout charts for the location of these pins.

On all parts with more than 8 pins, the TWI pins can be swapped to an alternate location.

On 2.0.0 and later, this is configured using the Wire.swap() or Wire.pins() methods. Both of them achieve the same thing, but differ in how you specify the set of pins to use. This should be called **before** Wire.begin().

`Wire.swap(1) or Wire.swap(0)` will set the the mapping to the alternate (1) or default (0) pins. It will return true if this is a valid option, and false if it is not (you don't need to check this, but it may be useful during development). If an invalid option is specified, it will be set to the default one.

`Wire.pins(SDA pin, SCL pin)` - this will set the mapping to whichever mapping has the specified pins as SDA and SCL. If this is not a valid mapping option, it will return false and set the mapping to the default. This uses more flash than Wire.swap(); that method is preferred.

In versions prior to 2.0.0, this was instead configured using the Tools -> I2C Pins submenu; this is set at compile time (reburning bootloader is not required). In these versions, you could see whether the alternate pins are in use by checking if TWIREMAP is #defined - you can, for example, use it to check whether correct options are selected and terminate compilation, so you can select the right option.

As of 1.1.9, courtesey of https://github.com/LordJakson, when the version of Wire.h supplied with megaTinyCore 1.1.9 and later in slave mode, it is now possible to respond to the general call (0x00) address as well. This is controlled by the optional second argument to Wire.begin(). If the argument is supplied amd true, general call broadcasts will also trigger the interrupt. This version also introduces a third optional argument, which is passed unaltered to the TWI0.SADDRMASK register. If the low bit is 0, and bits set 1 sill cause the I2C hardware to ignore that bit of the address (masked off bits will be treated as matching). If the low bit is 1, it will instead act as a second address that the device can respond to.

### PWM support
The core provides hardware PWM (analogWrite) support. On the 8-pin parts (412, 212, 402, 204), 4 PWM pins are available. On all other parts except the x16 and x17 series, 6 PWM pins are available, all driven by Timer A. The Type B timers cannot be used for additional PWM pins - their output pins are the same as those available with Timer A - however you can take them over if you need to generate PWM at different frequencies. See the pinout charts for a list of which pins support PWM.
The 3216,1616,816,416,3217,1617 and 817 have two additional PWM pins driven by Timer D (PC0 and PC1 - pins 10 and 11 on x16, 12 and 13 on x17). Timer D is an async timer, and the outputs can't be enabled or disabled without briefly stopping the timer. This results in a brief glitch on the other PWM pin (if it is currently outputting PWM), and doing so requires slightly longer - though the duration of this glitch is under 1 us. If TCD is used as the millis timer - which is the default on these parts as of 1.1.9 - this will result in millis losing that much time as well. Note that in versions prior to 1.1.9, these pins could not be used for PWM if TCD0 was used as the millis timekeeping source; that restriction was lifted with 1.1.9. This applies to digitalWrite() or analogWrite of 0 or 255 while it is currently outputting PWM, and analogWrite of 1~254 while the pin is not currently outputting PWM. This is a hardware limitation and cannot be further improved.

**Note that TCA0 (the type A timer) on all parts is configured in split mode to support the most PWM pins possible with analogWrite(). As of 1.1.6, and to a lesser extent other versions since 1.1.3 it has been made much easier to reconfigure TCA0 without messing up other functions of the core. If you want to reconfigure TCA0 for other purposes, please refer to the below guide**
#### [Taking over TCA0](megaavr/extras/TakingOverTCA0.md)

**For general information on the available timers and how they are used PWM and other functions, consult the guide:**
#### [Timers and megaTinyCore](megaavr/extras/PWMandTimers.md)

In versions prior to 1.1.7, reconfiguting the TCA0's prescaler adversely effected tone and servo functionality. This is no longer the case.

### NeoPixel (WS2812) support
The usual NeoPixel (WS2812) libraries have problems on these parts. This core includes two libraries for this, both of which are tightly based on the Adafruit_NeoPixel library. See the [tinyNeoPixel documentation](megaavr/extras/tinyNeoPixel.md) and included examples for more information. As of 2.0.1, these libraries are split into three versions specific to the part to eliminate the use of an additional submenu option.

### Tone Support
Support for tone() is provided on all parts using TCB0, unless TCB1 is present and TCB0 is set as millis source. This is like the standard tone() function; it does not support use of the hardware output compare to generate tones. See caveats below if using TCB0 or TCB1 for millis/micros settings.

### millis/micros Timekeeping Options
megaTinyCore provides the option to use any available timer on a part for the millis()/micros timekeeping, controlled by a Tools submenu - or it can be disabled entirely to save flash and allow use of all timer interrupts. By default, TCD0 will be used by on parts that have one - otherwise TCA0 will be used (in versions prior to 1.1.9, TCA0 was used by default on parts that could output PWM with TCD0 on pins not available for TCA0 PWM). All timers available on the parts can be used: TCA0, TCD0 (on parts that have it), TCB0, TCB1 (where present) and the RTC. Many of these - particularly the non-default options, involve tradeoffs. In brief, TCA0 is a very versatile timer that users often want to reconfigure, TCD0 loses a small amount of time when PWM is turned on or off on the two TCD0 PWM pins (10,11 on 20-pin parts, 12,13 on 24-pin parts), TCB0 conflicts with Servo and tone on parts that don't have TCB1, and when the RTC is used micros() is not available at all becuase the clock isn't fast enough. With these limitations in mind, the timer selection menu provides a way to move millis()/micro to the timer most appropriate for your needs.

For more information, on the hardware timers of the supported parts, and how they are used by megaTinyCore's built-in functionality, see the [Timers and megaTinyCore](megaavr/extras/PWMandTimers.md)

In recent versions, the timekeeping timer options have been undergoing frequent and significant improvements. It is recommended that all users - particularly those with demanding timer requirements - use 1.1.9 or later.

In versions prior to 1.1.9, with a 20/10/5 MHz system clock, micros() produced inaccurate results (in several ways), timer accuracy and consistency of micros results at 1 MHz was particularly poor. A summary of the changes to the timers when used for millis and micros is (available on google docs)[https://docs.google.com/spreadsheets/d/1W6XAChKxozjN87hF34xwsb6TCKHA1KMztx8wL_UbLmk/edit?usp=sharing]


#### RTC timer for millis
If the RTC is selected as the timer for millis timekeeping, micros will not be available. Additionally, this timer is configured to run while in STANDBY sleep mode. This has two important consequences: First, it will keep time while in sleep. Secondly, every 64 seconds, the RTC overflow interrupt will fire, waking the chip - thus, if you are using the RTC for millis and putting the part into sleep, you should declare a volatile global variable that you set in the ISR that is supposed to wake the part, eg 'volatile boolean ShouldWakeUp=0;' - set it to 1 in the ISR, and when you put the ATtiny to sleep, have it check this immediately after waking, going back to sleep if it's not set, and clearing it if it is, e.g.:

```cpp
void GoToSleep() {
    do {
        sleep_cpu();
    } while (!ShouldWakeUp)
    ShouldWakeUp=0;
}
```

This functionality will be made easier to use via the megaTinySleep library in a future version of the core.

This board package also supports using an external 32.768khz crystal as the clock source for the RTC (not supported on 8-pin parts). If this is used, make sure that the crystal is connected between the TOSC1 and TOSC2 pins (these are tbe same as the TX and RX pins), that nothing else is, that no excessively long wires or traces are connected to these pins, and that appropriate loading capacitors per crystal manufacturer datasheet are connected. Since the TOSC1 and TOSC2 pins are the same pins used for serial, you must use the alternate serial pins.

### ADC Support
These parts all have ADC channels available on most pins (11 pins on 24 and 20 pin parts, 9 on 14-pin parts and 5 on 8 pin parts - plus the one on the UPDI pin which is not normally usable), they can be read with analogRead() like on a normal AVR. While the An constants (ex, A0) are supported, and refer to the corresponding ADC channel (not the corresponding pin number), using these is deprecated - the recommended practice is to pass the digital pin number to analogRead(). Analog reference voltage can be selected as usual using analogReference(). Supported reference voltages are:
* VDD (Vcc/supply voltage - default)
* INTERNAL0V55
* INTERNAL1V1
* INTERNAL1V5
* INTERNAL2V5
* INTERNAL4V3 (alias of INTERNAL4V34)
* INTERNAL4V34
* EXTERNAL (1-series - not including 412/212 - only)

In addition to the pin numbers, as of 1.1.10, you can read from the following sources:
* ADC_INTREF (The internal reference - you can set it manually via VREF.CTRLA, or by calling analogReference(), followed by analogReference(VDD) - The internal reference register is not changed when switching back to VDD as reference)
* ADC_DAC0 (The value being output by DAC0, 1-series parts only)
* ADC_TEMPERATUIRE (The internal temperature sensor)

As of 2.0.0, we have taken full advantage of the improvements in the ADC on the megaAVR parts to improve the speed of analogRead() by more than a factor of three - the ADC clock which was - on the classic AVRs - constrained to the range 50~200kHz - on the megaavr parts can be cranked up as high as 1.5MHz. As of 2.0.0, we now use 1MHz at 16/8/4 MHz system clock, and 1.25 MHz at 20/10/5 MHz system clock. To compensate for the faster ADC clock, we set ADC0.SAMPCTRL to 14 to extend the sampling period from the default of 2 ADC clock cycles to 16 - providing the same sampling period as we did in previous versions, which should preserve the same accuracy when measuring high impedance signals. If you are measuring a lower impedance signal and need even faster analogRead() performance - or if you are measuring a high-impedance signal and need a longer sampling time, you can adjust that setting:

```
ADC0.SAMPCTRL=31; // maximum sampling length = 31+2 = 33 ADC clock cycles
ADC0.SAMPCTRL=0; //minimum sampling length = 0+2 = 2 ADC clock cycles
```

With the minimum sampling length, analogRead() speed would be approximately doubled from it's already-faster value.

**Note:** The 3217,1617,3216,1616, and 1614 have a second ADC, ADC1. On the 20 and 24-pin parts, these could be used to provide analogRead() on additional pins (it can also read from DAC2). Currently, there are no plans to implement this in the core due to the large number of currently available pins. Instead, it is recommended that users who wish to "take over" and ADC to use the advanced features available on the megaavr parts choose ADC1 for this purpose.

### DAC Support
The 1-series parts have an 8-bit DAC which can generate a real analog voltage (note that this provides very low current and can only be used as a voltage reference, it cannot be used to power other devices). This generates voltages between 0 and the selected VREF (which cannot be VCC, unfortunately). In 2.0.0 and later, set the DAC reference voltage via the DACReference() function - pass it one of the INTERNAL reference options listed under the ADC section above. In versions prior to 2.0.0, select the DAC VREF voltage from the Tools -> DAC Voltage Reference submenu. This voltage must be lower than Vcc to get the correct voltages. Call analogWrite() on the DAC pin to set the voltage to be output by the DAC. To turn off the DAC output, call digitalWrite() on that pin.

### Servo Support
This core provides a version of the Servo library which will select an appropriate timer (TCB0, except on the 3216, 3217, 1617, 1616 and 1614, where there is a Timer B 1 available; except on the aforementioned parts, tone cannot be used at the same time as the Servo library. If millis/micros is set to use TCB1 on those parts, servo will use TCB0 instead, making it incompatible with tone there as well). Servo output is better the higher the clock speed - when using servos, it is recommended to run at the highest frequency permitted by the operating voltage to minimize jitter.

**Warning** If you have installed a version of the Servo library to your <sketchbook>/libraries folder (including via library manager), the IDE will use that version of the library (which is not compatible with these parts) instead of the one supplied with megaTinyCore. As a workaround, a duplicate of the Servo library is included with a different name - to use it, `#include <Servo_megaTinyCore.h>` instead of `#include <Servo.h>`

### printf() support for "printable" class
Unlike the official board packages, but like many third party board packages, megaTinyCore includes the .printf() method for the printable class (used for Serial and many other libraries that have print() methods); this works like printf(), except that it outputs to the device in question; for example:
```cpp
Serial.printf("Milliseconds since start: %ld\n", millis());
```
Note that using this method will pull in just as much bloat as sprintf(), so it may be unsuitable on devices with small flash memory.

### Pin Interrupts
All pins can be used with attachInterrupt() and detachInterrupt(), on RISING, FALLING, CHANGE, or LOW. All pins can wake the chip from sleep on CHANGE or LOW. Pins marked as ASync Interrupt pins on the pinout chart can be used to wake from sleep on RISING and FALLING edge as well.

Advanced users can instead set up interrupts manually, ignoring attachInterrupt and manipulating the relevant port registers appropriately and defining the ISR with the ISR() macro - this will produce smaller code (using less flash and ram) and the ISRs will run faster as they don't have to check whether an interrupt is enabled for every pin on the port. For full information and example, see: [Pin Interrupts](megaavr/extras/PinInterrupts.md)

### Assembler Listing generation
Like ATTinyCore, Sketch -> Export compiled binary will generate an assembly listing in the sketch folder; this is particularly useful when attempting to reduce flash usage, as you can see how much flash is used by different functions.

### EESAVE configuration option
The EESAVE fuse can be controlled via the Tools -> Save EEPROM menu. If this is set to "EEPROM retained", when the board is erased during programming, the EEPROM will not be erased. If this is set to "EEPROM not retained", uploading a new sketch will clear out the EEPROM memory. You must do Burn Bootloader to apply this setting.
**WARNING** In megaTinyCore 1.0.6 and earlier, this setting is backwards - setting it to retained will not retain, and vise versa. This is corrected in 1.0.7 and later.

### BOD configuration options
These parts support many BOD trigger levels, with Disabled, Active, and Sampled operation options for when the chip is in Active and Sleep modes - Disabled uses the least power, Active uses the most, and Sampled is in the middle. See the datasheet for details on power consumption and the meaning of these options. You must do Burn Bootloader to apply this setting.

### Link-time Optimization (LTO) support
This core *always* uses Link Time Optimization to reduce flash usage - all versions of the compiler which support the 0-series and 1-series ATtiny parts also support LTO, so there is no need to make it optional as was done with ATtinyCore.

### Identifying menu options within sketch
It is often useful to identify what options are selected on the menus from within the sketch; this is particularly useful for verifying that you have selected the options you wrote the sketch for when you open it; however, note that as of 2.0.1, almost all of these options have been removed except for the millis timer option.

##### Millis timer
The option used for the millis/micros timekeeping is given by a define of the form USE_MILLIS_TIMERxx. Possible options are:
* USE_MILLIS_TIMERA0
* USE_MILLIS_TIMERB0
* USE_MILLIS_TIMERB1
* USE_MILLIS_TIMERD0
* USE_MILLIS_TIMERRTC
* DISABLE_MILLIS

##### Using to check that correct menu option is selected
If your sketch requires that the B0 is used as the millis timer

```
#ifndef MILLIS_USE_TIMERB0
#error "This sketch is written for use with TCB0 as the millis timing source"
#endif
```

### Identifying part family within sketch
When writing code that may be compiled for a variety of target chips (which is particularly easy on the megaAVR parts, as the peripherals are so similar), it is often useful to be able to identify which family of parts is being used, in order to, for example, select which pin numbers to use for each purpose. Each part is identified by a #define:
`__AVR_ATtiny***__` where *** is the part number (ex: `__AVR_ATtiny3216__`).

This core provides two additional #defines for part "families":
* `__AVR_ATtinyx**__` where ** is the last 2 digits of the part number (ex: `__AVR_ATtinyx16__`)
* `__AVR_ATtinyxy*__` where * is the last digit of the part number, (ex: `__AVR_ATtinyxy6__`).

This is just shorthand, for convenience - `#ifdef __AVR_ATtinyxy2__` is equivilent to `#if defined(__AVR_ATtiny212__) || defined(__AVR_ATtiny412__) || defined(__AVR_ATtiny202__) || defined(__AVR_ATtiny402__)`

Additionally, `MEGATINYCORE` will be defined whenever this core is in use.

# Bootloader (optiboot) Support
A new version of Optiboot (Optiboot-x) now runs on the Tiny0 and Tiny1 chips.  It's under 512 bytes, and works on all parts supported by megaTinyCore, allowing for a convenient workflow with the same serial connections used for both uploading code and debugging (like a normal Arduino Pro Mini).

To use the serial bootloader, select a board definition with (optiboot) after it (note - this might be cut off due to the width of the menu; the second set of board definitions are the optiboot ones).

If the chip you will be programming has not been bootloaded, connect your UPDI programmer, and the desired options for clock rate (20/10/5MHz vs 16/8/4/1MHz is all that matters here. The fuses set the base clock to 20MHz or 16MHz, but the prescaler is set at startup by the sketch - so if you "burn bootloader" with 20MHz selected, but upload sketch compiled for 10MHz, that's fine and will work), Brown Out Detect (BOD), Serial port pins for the bootloader, and whether to leave the UPDI pin configured as such, or reconfigure it as a reset pin, and select Tools -> "Burn Bootloader"
**WARNING: After doing "Burn Bootloader", if you set the UPDI pin to act as reset or IO, the chip cannot be reprogrammed except via the serial bootloader or using an HV UPDI programmer - it is strongly suggested to first burn the bootloader with the UPDI/Reset pin left as UPDI, verify that sketches can be uploaded, and only then "Burn Bootloader" with the UPDI/Reset pin set to act as Reset**

After this, connect a serial adapter to the serial pins (as well as ground and Vcc). On the megaavr ATtiny breakout boards which I sell on Tindie, a standard 6-pin "FTDI" header is provided for this that can be connected directly to many serial adapters.


If the UPDI/Reset pin option was set to UPDI or IO when bootloading, you must unplug/replug the board (or use a software reset - see note near start of readme; you can make an ersatz reset button this way) to enter the bootloader - after this the bootloader will be active for 8 seconds.

If the UPDI/Reset pin option was set to reset, you must reset the chip via the reset pin (or software reset to enter the bootloader, and the bootloader will run for 1 second before jumping to the application). This is the same as how optiboot works on classic AVR boards. The standard DTR-autoreset circuit is recommended (this is how boards I sell on Tindie with Optiboot preloaded are configured).

Serial uploads are all done at 115200 baud, regardless of speed or part.

### Autoreset circuit
If using the bootloader with with the UPDI pin set to reset, this is needed for uploading. It will reset the chip when serial connection is opened like on typical Arduino boards. Do not connect an autoreset circuit to the UPDI pin if it is not set to act as reset, as this will not reset the chip, and will just block UPDI programming.
* 1 Small signal diode (specifics aren't important, as long as it's approximately a standard diode)
* 1 0.1uF Ceramic Capacitor
* 1 10k Resistor
* An ATtiny megaavr with the bootloader installed on it (you can't do UPDI programming once the autoreset circuit is connected the the UPDI/Reset pin, even if that pin hasn't yet been set to act as reset, as the autoreset circuit will interfere with UPDI programming).

Connect the DTR pin of the serial adapter to one side of a 0.1uF ceramic capacitor.
Connect the other side of the capacitor to the Reset (formerly UPDI) pin.
Connect the 10k resistor between Reset and Vcc.
Connect the diode between Reset and Vcc with the band towards Vcc.

The breakout boards I sell on Tindie have a pair of pads on the back that can be bridged with solder to reversibly connect the on-board autoreset circuit to the UPDI/Reset pin. If you buy a breakout board from me which has the autoreset circuit, but which is not bootloaded, these pads will not have been connected - use a UPDI programmer to put the bootloader on it with your desired settings, then bridge the pads on the back to connect the auto-reset circuit. If you buy a breakout board from me with Optiboot preloaded, it will come set for 20/10/5MHz, BOD set to "sampled" mode with a threshold of 3.7v (for 5v boards) or 2.6v (for 3.3v boards) or disabled entirely (for no-regulator boardds), and if you ordered a board with the autoreset circuit, the UPDI pin will be set to work as reset (if you need different settings, please mention this when ordering - boards with optiboot preloaded are bootloaded only when the order is packed, so I can use whatever settings you specify).

Note that, if you have UPDI programming enabled, and desire the convenience of autoresetting to enter serial bootloader to upload sketches, or resetting the chip when entering the serial monitor, you can use the technique described near the top of the readme (under the section about there not being a dedicated reset pin) to make an ersatz reset pin that does a software reset when held low - this will enter the bootloader, so if your sketches set up an ersatz reset pin, you can use that to get into the bootloader (this isn't foolproof, because if your sketch chokes before it properly sets up the interrupt, or you choked and forgot to include the ersatz reset code in the last sketch you uploaded - but in this case you can enter bootloader by power cycling as described above). The autoreset circuit works the same way - just connect the parts as described above, only using your ersatz reset pin instead of the UPDI/Reset pin.

### A few caveats:
* The bootloader is at the beginning of memory, rather than at the end (where it was on older chips). Thus, the start of the application code must be 512b after the start of the memory - this is handled by the core, but you cannot upload a .hex file compiled with a non-optiboot board definition to an optiboot board definition and vise versa.
* **If you have set the UPDI/Reset pin to act as a reset pin, you can no longer program the part via UPDI** without using an HV programmer to reset the pin to act as UPDI.
* Currently, Optiboot_x resets the reset cause register after saving the contents in R2.
* The new chips have more than one option for Uart Pins. The option selected when you "burn bootloader" determines which version is used. If using the breakout boards I sell on Tindie, the first menu option uses the pins connected to the 6-pin FTDI-style serial header.
* When you "burn bootloader", the base oscillator frequency is set according to the selected clock speed, but the actual operating speed while running the sketch is set in the uploading code. If you initially set it to 16/8/4/1MHz, you may use any of those options when you upload your sketch and it will run at that speed; if you initially set it to 20/10/5MHz, you may use any of those options. If you wish to change between 16/8/4/1MHz and 20/10/5MHz, you must burn bootloader again - failure to do so will result in it running at the wrong speed, and all timing will be wrong.
* The "size" of the sketch as reported by avrdude during the upload process is 512b larger (the size of the bootloader) than the actual sketch size when a bootloader is in use. The fact that the bootloader goes at the start of the flash instead of the end confuses avrdude. The size displayed by the IDE when you "verify" a sketch is correct, the value that avrdude displays is not.

# Guides
### [Power Saving techniques and Sleep](https://github.com/SpenceKonde/megaTinyCore/blob/master/megaavr/extras/PowerSave.md)
### [Direct Port Manipulation](https://github.com/SpenceKonde/megaTinyCore/blob/master/megaavr/extras/DirectPortManipulation.md)
### [Pin Interrupts](https://github.com/SpenceKonde/megaTinyCore/blob/master/megaavr/extras/PinInterrupts.md)

# List of Tools sub-menus
* Tools -> Chip - sets the specific part within a selected family to compile for and upload to.
* Tools -> Clock Speed - sets the clock speed. You must burn bootloader after changing between 16/8/4/1MHz and 20/10/5MHz to apply the changes (ie, changing from 20MHz to 10MHz does not require burning bootloader, changing from 20MHz to 16MHz does). A virgin chip will be set to use the 20MHz internal oscillator as its base clock source, so 20/10/5MHz clock speed options should work without an initial "burn bootloader" - though we still recommend it to ensure that all other fuses are set to the values you expect.
* Tools -> Retain EEPROM - determines whether to save EEPROM when uploading a new sketch. You must burn bootloader after changing this to apply the changes. This option is not available on Optiboot board definitions - programming through the bootloader does not execute a chip erase function.
* Tools -> UART pins - All of these parts have the potential to remap the UART pins. This menu option sets which pins will be used as TX and RX by default. If Optiboot is being used, the selection from this menu when you "burn bootloader" will set which set of pins will be used by Optiboot. However, once the bootloader has been uploaded, you may change this if you need the sketch to use the other set of pins - the pins chosen when the bootloader was burned will still be used for uploads, but the current selection will be used by the sketch. **THIS MENU OPTION IS REMOVED IN 2.0.0 AND LATER** Instead, use the newly added Serial.pins() and Serial.swap() functions. See the section on Serial for more information.
* Tools -> B.O.D. Voltage - If Brown Out Detection is enabled, when Vcc falls below this voltage, the chip will be held in reset. You must burn bootloader after changing this to apply the changes. Take care that these threshold voltages are not exact - they may vary by as much as +/- 0.3v! (depending on threshold level - see electrical characteristics section of datasheet). Be sure that you do not select a BOD threshold voltage that could be triggered during programming, as this can prevent successful programming via UPDI (reported in #86).
* Tools -> UPDI/Reset pin - This menu is only available on board definitions that use the Optiboot bootloader. If set to UPDI, the pin will be left as the UPDI pin, there will be no hardware reset pin - to enter the bootloader, disconnect and reconnect power to the part, and upload within 8 seconds. If set to Reset, the pin will be configured to act as reset, like a classic AVR, but **UPDI programming will no longer work - you must use an HV programmer if you wish to reenable UPDI** - if the pin is set to reset, the version of Optiboot uploaded will jump straight to the application after a power-on reset, and will only enter the bootloader if reset by software reset or via the reset pin. The bootloader will also only wait 1 second for a sketch (ie, it behaves like optiboot does on classic AVR microcontrollers), instead of 8.
* Tools -> B.O.D. Mode (active) - Determines whether to enable Brown Out Detection when the chip is not sleeping. You must burn bootloader after changing this to apply the changes.
* Tools -> B.O.D. Mode (sleep) - Determines whether to enable Brown Out Detection when the chip is sleeping. You must burn bootloader after changing this to apply the changes.
* Tools -> DAC Reference Voltage - Determines the voltage reference for the DAC. This should be less than Vcc to get the right voltage. You do not need to use Burn Bootloader to apply changes to this menu.
* Tools -> tinyNeoPixel Port - If using the tinyNeoPixel library (see above), and you are running at 8 or 10 MHz, you must set this option to the port with the pin(s) you are using. Not present on the 8-pin parts, as they only have one port. If not using tinyNeoPixel library or running at 16MHz or more, this option can be ignored.
* Tools -> Voltage for UART Baud - Controls which oscillator error values will be used to maximize the accuracy of the UART baud rate generation - choose whether operating voltage is closer to 5v or 3v.
* Tools -> millis()/micros() - If set to enable (default), millis(), micros() and pulseInLong() will be available. If set to disable, these will not be available, Serial methods which take a timeout as an argument will not have an accurate timeout (though the actual time will be proportional to the timeout supplied); delay will still work. Disabling millis() and micros() saves flash, and eliminates the millis interrupt every 1-2ms; this is especially useful on the 8-pin parts which are extremely limited in flash. Depending on the part, options to force millis/micros onto specific timers are available. A #error will be shown upon compile if a specific timer is chosen but that timer does not exist on the part in question. If RTC is selected, micros() andpulseInLong() will not be available - only millis() will be.

# Known Compiler Bugs
* Sometimes a sketch which is too big to fit will, instead of generating a message saying that, result in the error: 'relocation truncated to fit: R_AVR_13_PCREL against symbol tablejump2'.
* At least as of the avr-gcc package 7.3.0-atmel3.6.1-arduino5 there is a bug in the `<avr/eeprom.h>` library's eeprom_is_ready() macro - it refers to registers that don't exist on the parts (this impacts the 4809 and it's ilk as used in Nano Every/Uno WiFi Rev. 2 as well). Attempting to use that macro will generate a error stating that `'NVM_STATUS' was not declared in this scope`. When a newer version of that package that corrects this issue is available, megaTinyCore will use it. In the meantime, `replace eeprom_is_ready()` with `bit_is_clear(NVMCTRL.STATUS,NVMCTRL_EEBUSY_bp)`. If writing a library or sketch that is intended to work on both megaavr and classic avr parts, you can test for effected parts using `#if defined(ARDUINO_ARCH_MEGAAVR)`


# Buying ATtiny megaAVR breakout boards
I sell breakout boards with regulator, UPDI header, and Serial header in my tindie shop, as well as the bare boards. Buying from my store helps support further development on the core, and is a great way to get started using these exciting new parts with Arduino.

### [ATtiny3217, 1607 assembled](https://www.tindie.com/products/17523/)
### [ATtiny3217/1617/817/417/1607/807 bare board](https://www.tindie.com/products/17613/)
### [ATtiny3216, 1606 assembled](https://www.tindie.com/products/17597/)
### [ATtiny3216/1616/816/416/1606/806/406 bare board](https://www.tindie.com/products/17614/)
### [ATtiny1614, 1604 assembled](https://www.tindie.com/products/17598/)
### [ATtiny1614/814/414/1604/804/404 bare board](https://www.tindie.com/products/17748/)
### [ATtiny412, 402 assembled](https://www.tindie.com/products/17685/)
### [ATtiny412/212/402/202 bare board](https://www.tindie.com/products/17749/)
