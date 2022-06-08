# eChallengeCoin 2020 Hack Guide

Before Defcon 28 (yeah, the one that "has been cancelled" :-)), [Bradán Lane Studio](https://www.tindie.com/products/bradanlane/echallengecoin-2020/) released this really interesting badge/coin/board, with a very funny adventure on it. You can find some more details [here](https://aosc.cc/).

<img src="https://github.com/cecio/eChallengeCoin-2020/blob/main/Pictures/thecoin.png" alt="thecoin" style="zoom:67%;" />

I started to play with it, but I found myself too dumb to solve the challenges...so I tried to follow another way, which is more familiar to me...reversing :-). This is a little guide on how I did it. 

I'm intentionally not giving you details on the challenge solutions and I don't want to spoil too much. If you want to solve them in the 'hacky' way, follow the instructions and put some effort in it. And since I found the following sentences buried in the data during my analysis, I assume it was expected and allowed :-)

```
you did not solve the challenges using the clues in the story and you just hacked the eChallengeCoin. That's OK too!
```

```
and you just hacked the eChallengeCoin. That's OK too!
```

You can follow this guide while waiting the 2021 version!

## Tools

I used several tools in this process, some of them were not really useful. At the end this is a list of what helped me:

- An **Arduino Uno** used to dump the firmware and the EEPROM
- [simavr](https://github.com/buserror/simavr) to emulate the board
- **gdb** (for AVR) for the dynamic analysis
- [Cutter](https://cutter.re/) for the static analysis

## Step 1: a look into the Hardware

The eChallngeCoin has a very cool look & feel. But it does not expose anything in its initial "configuration". If you remove the 3D printed cover on the back, you expose the "core" of the board. You can see an `ATMega328PB` MCU, some LEDs and resistors.

As you can see the board exposes the PINs/interfaces on the edge of the coin (let's call them PADs). Some of them (4) are not labelled, or better they are, but  with some cryptic tags: M1, M2, K1 and X1. This attracted my attention. With the help of a multimeter and the MCU datasheet, I mapped the 4 unlabelled PADs as follow:

<img src="https://github.com/cecio/eChallengeCoin-2020/blob/main/Pictures/echallenge_coin_ICSP.png" alt="echallenge_coin_ICSP" style="zoom:67%;" />

- M1: MISO
- M2: MOSI
- K1: Clock
- X1: RST

This look very promising, since this seems to be SPI port connections. If I can attach a "programmer" to this, may be I can dump the content of the flash.

## Step 2: dumping EEPROM and FLASH

**WARNING**: *before following these steps, I suggest to remove the battery from the eChallengeCoin. And, as a general rule, if you do something wrong, you may brick/destroy your boards. Be careful, you are responsible of your actions!*

I did this with an **Arduino UNO**, but you can probably use also some other versions, or even another ISP programmer.

To set up the Arduino, you can follow this [guide](https://www.arduino.cc/en/Tutorial/BuiltInExamples/ArduinoISP). Once you uploaded the proper firmware, you can just connect the MISO, MOSI and SCK PINs to the eChallengeCoin PADs. Remember to connect the RST PAD to PIN **10** of the Arduino. You can also use the 3.3v and GND PINs to power the eChallengeCoin (**do not attach it to the 5v PIN!**).

Once you have your Arduino attached to the USB port, and the eChallengeCoin attached to the Arduino, you can use `avrdude` to dump everything.

This saves the `flash` (in two different formats, Intel Hex and Raw):

```
avrdude -q -patmega328pb -cstk500v1 -P /dev/ttyACM0 -b 19200 -D -Uflash:r:flash.bin:r
avrdude -q -patmega328pb -cstk500v1 -P /dev/ttyACM0 -b 19200 -D -Uflash:r:flash.hex:i
```

This saves the `EEPROM` (in two different formats, Intel Hex and Raw):

```
avrdude -q -patmega328pb -cstk500v1 -P /dev/ttyACM0 -b 19200 -D -Ueeprom:r:eeprom.bin:r
avrdude -q -patmega328pb -cstk500v1 -P /dev/ttyACM0 -b 19200 -D -Ueeprom:r:eeprom.hex:i
```

I dumped in both the format because I'll need them for the emulator and for the analysis.

The first thing I did just after the dump, was to open the files in an Hex Editor...but no way...no clear-text in them, except for few strings. We need to do more work!

## Step 3: Setup the environment

So, we need to use our reversing skills. I opted for **Cutter** to manage the static analysis. But I don't want to do everything statically, so I decided to use **simavr** to emulate locally the board and even attach a debugger to it.

Some time ago I wrote a quick article on [Paged Out!](https://pagedout.institute/) magazine on how to setup an **AVR** debug environment. Look at Issue #1 page 8. BTW, read also all the other articles of the magazine, you will enjoy them.

When you have your **simavr** and **gdb** ready, follow these steps:

- move in **simavr** folder and start the **simduino**: `./obj-x86_64-linux-gnu/simduino.elf -d`

- start **gdb**, attach to the debugger (`target remote localhost:1234`) and press **c** to continue execution

- upload **EEPROM** and **FLASH** in the emulator:

  `avrdude -p m328p -c arduino -P /tmp/simavr-uart0 -U eeprom:w:eeprom.hex`

  `avrdude -p m328p -c arduino -P /tmp/simavr-uart0 -U flash:w:flash.hex`

- you can now use your favourite terminal emulator to attach to `/tmp/simavr-uart0` and see the eChallengeCoin output

Now everything is ready to start the analysis.

## Step 4: the analysis

There are a lot of things we can investigate. For example the **EEPROM** dump

![eeprom](https://github.com/cecio/eChallengeCoin-2020/blob/main/Pictures/eeprom.png)

Most of the bytes seems to be unused, but few of them are set to some values. If you play with them you can understand what they control or represent. I didn't investigated all of them, you can do your tests.

**Hint**: if you want tamper it, you should do it on the Intel Hex format dump, since this is the format used by the emulator. The last two chars of each line, are the checksum. If you modify values in the line, you need to update the checksum. You can use the following Python code:

```
# Checksum: two complement of the sum of all bytes
hexline = ':2000000016FFFFFF50180001000100FFFFFFFFFF00FFFFFFFFFFFFFFFFFFFFFFFFFFFFFF'
hex(abs((sum([int(hexline[i:i+2],16) for i in range(1,len(hexline), 2)])& 0xFF) - (1<<8)))
```

**Hint**: I had to tamper a bit with the file to have everything running in the emulator.

Then, let's move to the code. I used a combination of static (**Cutter**) and dynamic analysis. It required some time to figure out the general picture, it was not so easy. You can explore a lot of things, but since I told you I don't want to spoil to much, I'll give you here a snippet of the string decryption/decompression routine isolated into the AVR code

<img src="https://github.com/cecio/eChallengeCoin-2020/blob/main/Pictures/cutter_dec.png" alt="cutter_dec" style="zoom:75%;" />

Placing breakpoints in the proper places (you should know how to do it if you have read everything until now), you should be able to find your way...

<img src="https://github.com/cecio/eChallengeCoin-2020/blob/main/Pictures/gdb_break.png" alt="gdb_break" style="zoom:75%;" />

Now that I did the heavy lifting, it's your turn!

## Step 5: some code

This is just a quick Python implementation of the decryption/decompression routine, if you want to use it:

```
def dec_buffer(buffer):
    i = 1
    oldb = 0
    dec = ''
    for b in buffer:
        
        tmpchr = chr((( oldb + b >> i ) & 0xFF) & 0x7F)
        if tmpchr.isprintable():
            dec += tmpchr
        else:
            break
        oldb = ( b << 8 )
        i += 1
    
        if i == 8:
            dec += chr(b & 0x7F) 
            i = 1
            oldb = 0
    
    print(dec)
```



## Wrap up

It was really interesting to look into this. I intentionally left some open questions in this tutorial...you know, it's a Challenge Coin...so...this is just a different set of challenges...call it a bonus track ;-)

If you have any question on this, you can reach me out on Twitter: @red5heep 

Happy hacking!

