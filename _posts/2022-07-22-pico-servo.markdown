---
layout: post
title:  "Controlling servo motors using Raspberry Pi Pico"
date:   2022-07-22 12:00:00 -0500
categories: projects
---

I recently picked up the new Raspberry Pi Pico microcontroller, and I'm having fun messing around with it. My first experiences with microcontollers were Arduino boards. Back in university, most of the computer science-related clubs I was in involved me writing software for Arduino microcontrollers.
Coming from the Arduino world, with all its convinient built-in libraries, I was used to interfacing with devices with only a couple lines of code. For PWM servos in particular, the code looked something like this:

```
Servo my_servo;
...
my_servo.attach(<pin>);
...
my_servo.write(<angle>);
```

Writing code for the Pico is immedately much more hands on, closer to the metal, etc. In fact, I was surprised that there was not any built in servo libraries at all. Looking around online, I couldn't find anyone who had made one for the pico. I decided to build my own.

# Servos, how do they work?

I won't go too much into detail, as there are better resources out there. Basically on the signal wire, a square wave oscillates at 50Hz. The angle of the servo is encoded in the pulse width, or how long each cycle of the wave is at a high voltage. For most analog servos a pulse width of 1ms is 0 degrees, and 2ms is 180 degrees. That seems easy enough, in theory.

![](/assets/images/servo1.png)

# PWM on the Pico

PWM on the Pico is much more involved than on Arduino. The Pico has 8 "slices", each with 2 channels, for a total of 16 PWM outputs.

The frequency and duty time of a PWM output cannot be set directly. Instead, clock division, wrap, and channel level are the three parameters that determine frequency and duty cycle.

### wrap and level

I like to imagine the `wrap` as some array of cells 
<br>`[ ][ ][ ][ ][ ][ ][ ][ ]`
<br>and `level` as how many cells are filled. So a wrap of 8 and a level of 3 produces
<br>`[x][x][x][ ][ ][ ][ ][ ]`.
<br>If you squint at
<br>`[x][x][x][ ][ ][ ][ ][ ]`
<br>it almost looks like a square wave `---_____`.

So with `wrap` and `level` combined, the duty cycle can be set, but how about the frequency?

### clkdiv

The Pico will set the PWM output based on the wrap and level by stepping through each "cell" of level, and setting it high or low based on wrap. How frequently the Pico takes these steps is determined by the clock input and `clkdiv`. The formula for the PWM frequency is
```
pwm_freq_hz = clock_hz / (clkdiv * (wrap + 1))
```
It's more useful to solve for clkdiv, since for servos `pwm_freq_hz = 50` and wrap is our resolution, so we'll say `wrap = 10000`. After rearranging
```
clkdiv = clock_hz / (pwm_freq_hz * (wrap + 1))
```

### Putting it all together

To `.attach()` a servo on the Pico, the following operations are performed:
```
gpio_set_function(<pin>, GPIO_FUNC_PWM);
pwm_config config;
pwm_config_set_wrap(&config, <wrap>);
pwm_config_set_clkdiv(&config, <clkdiv>);
pwm_init(<slice_num>, &config, true);
```
The following will write a value to that PWM channel.
```
pwm_set_chan_level(<slice>, <channel>, <level>);
```

# The API

My library abstracts all this PWM stuff away, providing a familiar interface:
```
servo_init();
servo_clock_auto();

servo_attach(<pin>);

for (;;) {
    servo_move_to(<pin>, 0);
    sleep_ms(500);
    servo_move_to(<pin>, 180);
    sleep_ms(500);
}
```

# Under the hood

In addition to taking care of all the configuration, tracking pins and slices and translating Euler angles to levels, the library double-buffers calls to `servo_move_to()`.
Double buffering in combination with an interrupt that runs when a PWM channel reaches the end of a `wrap`, new servo angles are only written at the end of a cycle. 
This prevents any jitter when changing the angle of the servo.

# Conclusion

Writing my own server library for a new microcontroller left me with a deeper understanding of PWM and hardware interrupts, and a renewed respect for hardware programmers. This was not an easy project to debug, especially without any measurement equipment such as an oscilliscope.

I hope you find this library useful. All of the source code can be found at<br>
[github.com/irishpatrick/pico-servo](https://github.com/irishpatrick/pico-servo).