---
title:  "Timing attack on arduino"
author: eax
date:   2015-04-10 14:00:00
tags:
- reverse
- hardware
- electronic
- side channel
- timing attack

---

I like Side Channels Attacks. It's a way to gather some information about your target (software, hardware device, web site) by a way (the channel) not directly related with the primary task of your target. These channels are unintentional consequences, or artifact, of the primary task.

## Let's give some examples.

You are in the same room as your target. The person is writing some text on his laptop and you want to know what it is. With a microphone, you record the audio for a couple of minutes. Guess what? We can use the noise made by pushing keys on a keyboard to recover what is being typed.
Here, the primary task of your keyboard is to allow you to type text. The noise made by typing on it is an unintentional consequence.


Every year a secret party is happening between some cool people you want to meet. Only the ones with the password can enter. And... You don't have this password. The good thing is that the host have a very bad memory, so just behind the front door, a screen has been hang up to display the party password in large black letters. Guess what? Screens are emitting electromagnetic leakage that can allow you to reconstruct the displayed image behind the door with some cheap device and some programming skills.
Here again, the electromagnetic radiation are artifact of the image being displayed.

## Overview

In this post, I will explain a basic side channel attack: Timing attack.
In Time attack, the channel that leak information is the time necessary to accomplish a task.

Let's imagine some door keypad:

 * The password is 6 digits long.
 * There is no "Enter key", the password is checked as soon as 6 keys has been pressed
 * Each key press make a led blink
 * When the password is wrong, a red led light up.
 * There is no trial limitation

How does it work? Well, we will assume that each key press is added on a buffer, and when the six digits has been typed, a routine is called to check this password against the one stored in memory. If this is the good password, the door unlock, otherwise, a red led blink.

## Assuming

We will assume that the check routine look like this:

```c
int check(char *input)
{
  int i;

  for (i = 0 ; i < 6 ; i++)
    if (input[i] != in_memory_password[i])
      return 0;
  return 1;
}
```

The C version is too high level to known exactly what will happen. So there is an ASM version of it: (AVR)

{% highlight asm linenos %}
ldi	r30, 0x09
ldi	r31, 0x01
movw	r26, r24
ld	r19, X+
movw	r24, r26
ld	r18, Z+
cpse	r19, r18    # Compare, Skip if Equal
rjmp	.+14		# jmp to line 16
ldi	r27, 0x01
cpi	r30, 0x0F
cpc	r31, r27
brne	.-20		# if eaqual, jmp to line 3
ldi	r24, 0x01
ldi	r25, 0x00
ret
ldi	r24, 0x00
ldi	r25, 0x00
ret
{% endhighlight %}


Let's see what happen if the password is 424344 and you enter 123456:

After 7 instructions, the first comparison occur:

{% highlight asm linenos linenostart=7 %}
cpse    r19, r18 # Compare, Skip if Equal
rjmp	.+14       # jmp to line 16
{% endhighlight %}

If the registers are equal, the next instruction is skipped. Here `r18` contain '4' and `r19` contain '1'. So the next instruction is not skipped and we jump at line 16. We set the return registers to 0 and return.

With a bad password, 11 instructions have been executed on the routine.
Now if we have only the first character good, the `rjmp` is skipped, and 9 more instructions are executed before the next `cpse`, and so one.

 
| Number of good characters|Number of instructions|
| -------------------------+---------------------|
| 0|11|
| 1|20|
| 2|29|
| 3|38|
| 4|47|
| 5|56|
| 6|68|

Last row is 68 and not 65 because there is 3 more instructions to do if it's the good password.
Okay, so more instruction mean more time, here his our time attack! Let's take a timer and measure it! Well... Kind of.

## Demo time!

To simulate the system, I used an arduino. I didn't have a keypad, so I just use the serial input from the computer. And as a timer, let's use an oscilloscope.

Here is the C++ code:

```c++
const char *in_memory_password = "424344";
const int led_pin = 13; 

void setup() {
  Serial.begin(9600);
  pinMode(led_pin, OUTPUT);
  digitalWrite(led_pin, LOW);
}

int check(char *input)
{
  int i;

  for (i = 0 ; i < 6 ; i++)
    if (input[i] != in_memory_password[i])
      return 0;
  return 1;
}

void loop() {
  char buf[32];
  int r;
 
  Serial.setTimeout(10);
  while (!Serial.available())
    delay(10);
  r = Serial.readBytes(buf, 31);
  buf[r] = 0;
  
  digitalWrite(led_pin, HIGH);
  r = check(buf);
  digitalWrite(led_pin, LOW);
    
  if (r)
    Serial.println("Good password");
  else
    Serial.println("Wrong password");
}
```

For the demo, a led is turned on before the check function, and turned off just after.
It's just to be simpler, but it could have been done with a beep on each key + a beep after the check.

To measure the time needed for the check function, I just hook up a probe on the pin 13 (led).

We said earlier than the password is 424344.
This is what we can see when we type 123456:

![input: 123456](/assets/2015-04-06-timing-attack-on-arduino/1.png)

Here the first comparison fail, and these 6.48us are corresponding to the 11 instructions. (Actually more since the code to turn on/off the led is consuming some time too)

So what happen if the first character of the input correspond to the one in the password?

![input: 400000](/assets/2015-04-06-timing-attack-on-arduino/4.png)

Ok so now, it take 0.8us more than the previous input. We guessed our first character!

Ok, yes, we knew the password, so there is no fun.

This time, let have a python script sending USBTMC command to the oscilloscope and talk in serial with the arduino to bruteforce the password!

Sound a little bit funnier now, right?

```python
#!/usr/bin/env python3

import string
import numpy
import serial
import matplotlib.pyplot as plot
import itertools
import instrument
from multiprocessing import Process, Queue
import time

timescale = 2.e-06
timeoffset = 10.e-06
vscale = 2
trig_lvl = 4

o = instrument.RigolScope("/dev/usbtmc0")

def setup_oscilo():
    o.write(":TIM:SCAL %f" % timescale)
    o.write(":TIM:OFFS %f" % timeoffset)
    o.write(":CHAN1:SCAL %f" % vscale)
    o.write(":TRIG:EDG:SOUR CHAN1")
    o.write(":TRIG:EDG:LEV %d" % trig_lvl)

def single_triger():
    o.write(":SINGLE")
    while o.wr(":TRIG:STAT?", 20) != b"WAIT\n":
        time.sleep(0.1)

def mesure_time(q):
    while o.wr(":TRIG:STAT?", 20) != b"STOP\n":
        time.sleep(0.1)
    o.write(":WAV:POIN:MODE NOR")
    o.write(":WAV:DATA? CHAN1")
    rawdata = o.read(9000)

    data = numpy.frombuffer(rawdata, 'B')
    l = len(data)
    data = data - (16.*8)

    # It should be 16 (nb of division on the scope).
    # But 12 works better I don't known why...
    data /= 12.
    
    elem_high = [x for x in filter(lambda x: x > trig_lvl, data)]
    res = len(elem_high)/50.
    q.put(res)

if __name__ == "__main__":
    sr = serial.Serial('/dev/ttyUSB0', 9600)
    setup_oscilo()

    pwd = ""
    last = ""
    while last != b"Good password":
        longest = (None, -1)

        for c in string.digits:
            print("Trying [%s]" % (pwd + c))

            q = Queue()
            p = Process(target=mesure_time, args=(q,))
            single_triger()
            sr.write(bytes(pwd + c, "utf8"))
            p.start()

            res = q.get()
            print("Time: %f" % res)
            if res > longest[1]:
                longest = (c, res)
            p.join()
            last = sr.readline().strip()
            if last == b"Good password":
                break

        pwd += longest[0]

    print("Found password! [%s]" % pwd)
```
(You can find the instrument class in the links section bellow)

Enjoy the password cracking:

[![asciicast](https://asciinema.org/a/18588.png)](https://asciinema.org/a/18588)

Only 55 try and 26 second and your in!

For the record, there is the screen for each step:

100000:

![input: 1](/assets/2015-04-06-timing-attack-on-arduino/1.png)

400000:

![input: 4](/assets/2015-04-06-timing-attack-on-arduino/4.png)

420000:

![input: 42](/assets/2015-04-06-timing-attack-on-arduino/42.png)[![asciicast](https://asciinema.org/a/18586.png)](https://asciinema.org/a/18586)

424000:

![input: 424](/assets/2015-04-06-timing-attack-on-arduino/424.png)

424300:

![input: 4243](/assets/2015-04-06-timing-attack-on-arduino/4243.png)

424340:

![input: 42434](/assets/2015-04-06-timing-attack-on-arduino/42434.png)


424344:

![input: 424344](/assets/2015-04-06-timing-attack-on-arduino/424344.png)


It was my introduction to side channel attack and particularly to time attack. I might do other post on side channel. Maybe about power analysis on this exact same code.

## Links

- [RSA Key Extraction via Low-Bandwidth Acoustic Cryptanalysis](http://www.cs.tau.ac.il/~tromer/papers/acoustic-20131218.pdf)
- [DEF CON 21 - Melissa Elliott - Noise Floor Exploring Unintentional Radio Emissions](https://www.youtube.com/watch?v=5N1C3WB8c0o)
- [Defcon 17 - Sniff Keystrokes With Lasers/Voltmeters - Side Channel Attacks Using Optical Sampling](https://www.youtube.com/watch?v=9zq9DQAbWmU)
- [python rigole oscilloscope](http://www.cibomahto.com/2010/04/controlling-a-rigol-oscilloscope-using-linux-and-python/)