---
permalink: /eeprom_to_lcd
layout: page
title: EEPROM to LCD
page_title: Output to LCD from EEPROM
---

![](assets/imgs/bare_metal_lcd.jpg)

In this project I created a circuit that outputs data stored in an EEPROM to a 16x2 LCD screen. There are five main components to the circuit, the clock, enable counter, address counter, EEPROM, and the LCD. This page details how each component works togather and a video is shown below of the circuit in action.

## Clock

The clock is a 555 timer configured in astable mode, both the Address and Enable counter run off this clock signal. The frequency is controlled by a potentiometer and can be seen by the white LED. This clock configuration was taken from [this]("https://www.youtube.com/watch?v=kRlSFm519Bo") Ben Eater video which is a great resource for understanding how this IC functions. For our purposes the clock provides our circuit with a 5v peak to peak square wave that our counter ICs run off of. In this configuration with a 10uf capacitor and two 330 ohm resistors alongside a 100k ohm potentiometer the clock can range from .71 Hz to 140 Hz.

## Address Counter

The chip used for the address counter is the CD4060B binary counter, a functional diagram of the IC is shown below. Pins 9, 10, and 11 can be used to configure the internal clock as seen in <a href="https://www.youtube.com/watch?v=tE0KNMN6244">this</a> video. In our case we are using the external clock mentioned above. This IC functions as a standard counter, on the negative edge of the clock pulse the counter advances. The first three bits of this counter are not mapped to pins on the IC, so the fourth bits is the lowest available. For timing reasons furthur discussed in the enable counter section the address counter uses the fifth bit of the ICs counter as its starting bit. This means that every 16 negative clock edges the address counter advances, which can be observed in the video below.

![](assets/imgs/timer_functional_diagram.png)

## Enable Counter

The enable pin tells the LCD when to take in data from the data pins. In the timing diagram shown below we can see that when the enable pin goes from high to low the data on the data pins will be taken in, this data can be instructions, or it can be values to display.

![](assets/imgs/lcd_timing_datasheet.png)

To achieve this timing I run the clock signal through a xor gate acting as a not gate, and run the enable counter from this new signals. So the address counter increments on the negative edge of the clock while the enable counter increments on the positive edge. The timings can be seen in the graph below. Before the first address bit can change the enable pin can toggle on and off effectively capturing the data before the address increments. The enable counter is shown by the green LED.

![](assets/imgs/lcd_timing_written.png)

## EEPROM

I used the CAT28C16A EEPROM to store the values needed to send to the LCD, this includes the values to initialize the LCD and the ascii values of the message "hello predo". I flashed the data to the EEPROM using the following circuit.

![](assets/imgs/flashing_circuit.jpg)

To write to the EEPROM we need to select the write mode using the chip enable(CE), output enable(OE),  and write enable(WE) pins, controlled by the three far left buttons on the bread board. Write mode is  entered when CE is low and OE is high. Data, given by the 8 pins on the right, is latched to the current address, given by the 5 pins in the middle, on the rising edge of the WE pin. I used the LCD in 4-bit mode so I only used 5 bits of the EEPROMs storage, 4 bits for the data pins and 1 bit to indicate instructions or value to display.

## LCD

I used a 1602a 16x2 LCD in 4bit mode. The RS pin indicates whether the value on the data pins are instructions or values to be displayed. A list of the values I used are shown below, and can be seen by the yellow LEDs in the video.

| Address | RS | D7 | D6 | D5 | D4 | Description |
| ------- | -- | -- | -- | -- | -- | ----------- |
| 0x0     | 0  | 0  | 0  | 1  | 0  | 4-bit Mode  |
| 0x1     | 0  | 0  | 0  | 0  | 0  | Clear Display |
| 0x2     | 0  | 0  | 0  | 0  | 0  | Clear Display |
| 0x3     | 0  | 0  | 0  | 0  | 0  | Return Home |
| 0x4     | 0  | 0  | 0  | 1  | 0  | Return Home |
| 0x5     | 0  | 0  | 0  | 1  | 0  | Display On, No Cursor |
| 0x6     | 0  | 1  | 1  | 0  | 0  | Display On, No Cursor |
| 0xX     | 1  | X  | X  | X  | X  | ASCII Value |
| 0xX     | 1  | X  | X  | X  | X  | ASCII Value |

The video below shows the circuit working, the ICs from top-left to the bottom-right are the address counter([CD4060BE](assets/datasheets/CD4060BE.pdf), the EEPROM([CAT28C16AP](assets/datasheets/CAT28C16AP.pdf)), the XOR gate([MC14070BCP](assets/datasheets/MC14070BCP.pdf)), the enable counter(([CD4060BE](assets/datasheets/CD4060BE.pdf)), and the clock([NE555P](assets/datasheets/NE555P.pdf)), and the LCD([1602A](assets/datasheets/1602A.pdf). The button controls the reset pin on the counters and the potentiometer controls the frequency of the clock.

[![eeprom_to_lcd](https://img.youtube.com/vi/Fb_cGUgnh1k/0.jpg)](https://www.youtube.com/watch?v=Fb_cGUgnh1k)
