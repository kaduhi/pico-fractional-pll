# pico-fractional-pll library
Make the RP2040 internal PLL a real "fractional" PLL (or, maybe more like "Pseudo Fractional" PLL)

## Background
We all know the RP2040 has two PLLs, and they only have an integer feedback divider so only specific frequencies can be generated. This means, you cannot use the PLL to generate radio signals meaningfully. Well, you are still able to generate Morse Code (by turning PLL on and off, it is still a digital signal), but you cannot tune the frequency of the RF signal.

However, one [genius guy](https://github.com/RPiks) came up with a great idea in 2023. He used PIO and overclocking technique to generate ANY frequency!!

[pico-hf-oscillator library](https://github.com/RPiks/pico-hf-oscillator)

[sample use case (WSPR transmitter)](https://github.com/RPiks/pico-WSPR-tx)

After I read [this article](https://hackaday.com/2023/12/03/pico-wspr-tx-does-it-in-software/), even before I cloned his repo, I started thinking how this genius achieved such an "impossible" thing. Yes, I'm one of many who uses the world famous [Si5351A](https://www.skyworksinc.com/en/Products/Timing/CMOS-Clock-Generators/Si5351A-B-GT) heavily for my [open source / open hardware Pico Balloon Tracker PCB](https://github.com/kaduhi/sf-hab_rp2040_picoballoon_tracker_pcb_gen1) because I believed there is no way the RP2040 could generate RF signals.

I came up with several different ideas. Yes, I had believed it was impossible, but now that I knew it was possible. It's so much easier to think.

Then, I check the answer. I cloned the repo and read the code. GOT IT! PIO! OVERCLOCKING! LOVE IT!! But even after seeing the answer, I still couldn't believe it can generate the WSPR signal that requires sub-Hz precision...

How about my other ideas? such as using the same technique that [some guy used with the original Raspberry Pi (not a Pico) DMA to generate the WSPR signal](https://en.wikipedia.org/wiki/WSPR_(amateur_radio_software)#/media/File:WsprryPi.JPG) a long time ago. OK, it might work, but still someone else's idea. So, the last one is more unique. Yes, go back to the RP2040's PLL again...

## How it works
OK, here is the idea:

**Use the PLL, but toggling the integer feedback divider between two values very quickly, faster than the LPF in the PLL feedback loop**

Then write a simple test code for a proof of concept:

```
  for (;;) {
    pll_sys->fbdiv_int = 114;
    pll_sys->fbdiv_int = 113;
  }
```

and indeed, this generated some frequency in between! (with some nasty harmonics...) Oh man, I just hacked the PLL!!

Then I use the [PDM](https://en.wikipedia.org/wiki/Pulse-density_modulation) technique to change the ratio of those two values. I checked the waterfall screen of my HF transceiver, it's drawing a diagonal line! Wow...

After that, I've tried many, many different ways to implement the PDM and improve the quality / stability of the signals:

- Use PWM block to generate the repeated interrupt
- Use a timer alarm to generate the repeated interrupt
- Use a M0's tick counter to generate the repeated exception
- Use assembly code to adjust the timing
- Use Core1 as a dedicated core for the PDM
- Use no interrupts, loop in assembly code with a lot of NOPs
- Adjust the timing of writing to the fbdiv_int register (relative to the XO clock)
- ...

I spent a few weeks to improve the quality of the generated signals. Its **timing** is very crtical!! And sometimes I couldn't understand WHAT'S GOING ON.
During testing, I saw "Generic Radio Waves from Aliens"-like waterfall patterns a lot! Yes, it looks like the signal is alive, and trying to tell me the truth of the universe...

Then, I lost interest. Just abandoned my code for 11 months...

## Good and Bad
Here is the good part and bad part of this library:

**Good:**

- Works on relatively low sys_clock frequency (no overclocking needed)
- Power consumption is relatively low (only the PLL is running on very high frequency)

**Bad:**

- Occupies Core1 (can't use for other purposes)
- The generated signal is not always clean
  - Sometimes a lot of harmonics (please use a Low Pass Filter, **NEVER attached an antenna wire directly to the GPIO port!!**)
  - If you happen to have a good RF spectrum analyzer, please check the output signal of this library... the post to YouTube, PLEASE!

## Limitations
This library still has some limitations:

- For now only works on 48MHz clk_sys frequency (uses the pll_usb for clk_sys)
  - USB works fine
- You need to specify the frequency range when you initialize this library
  - Some frequency range cannot be within the two integer divider values, so cannot be used
  - Lower the frequency will narrower the possible working frequency range
- Not tested all frequencies

## Some Specifications

- A GPIO port for the generated signal can be changed (GPIO 21, 23, 24 or 25, only GPIO 21 is available on Raspberry Pi Pico board and other pins are assiged to other purposes)
- PDM frequency is 1MHz (toggling the two fbdiv_int values for 1,000,000 times per second)
- Should be able to generate any frequency up to around 160MHz
  - I have tried to generate APRS signal on 440MHz, no signals came out...
- Uses the TIMER ALARM0 (can be changed to other alarm number)
- Uses the Cortex-M0+ tick counter with exception

## How to use this library

- Make sure the clk_sys is 48MHz
- Make sure the pll_sys is unused
- Make sure the Core1 is unused

The beginning of your main function should looks like:

```
int main(void) {
  set_sys_clock_48mhz();  // this will free the pll_sys, and set the source of the clk_sys to 48MHz pll_usb

  :
  :
```

And this is a simple *frequency sweeper* example:

```
void main(void) {
  set_sys_clock_48mhz();

  uint32_t freq_low = 87900000; // 87.9MHz
  uint32_t freq_high = 87915000; // 87.915MHz
  if (pico_fractional_pll_init(pll_sys, CLOCK_GPOUT0_PIN, freq_low, freq_high, GPIO_DRIVE_STRENGTH_12MA, GPIO_SLEW_RATE_FAST) != 0) {
    // ahhhh, the specified frequency range (freq_low ~ freq_high) cannot be within the two PLL divider values
    // therefore, the pico_fractional_pll cannot work!
    return;
  }

  // sweep the 15kHz range for 15 seconds
  for (uint32_t freq = freq_low; freq <= freq_high; freq++) {
    pico_fractional_pll_set_freq_u32(freq);
    sleep_ms(1);
  }

  pico_fractional_pll_deinit();
}
```

## Example Use Case

I have ported the [USB Sound Card](https://github.com/raspberrypi/pico-playground/tree/master/apps/usb_sound_card) demo from Raspberry Pi for this library, so anyone can use a Raspberry Pi Pico board as a **FM wireless transmitter** without modifying the board! "Well, I no longer own any AM/FM radio, so this example is useless!" really? check your car!

You should be able to build and download the demo by following steps below:

```
mkdir pico_fm_transmitter
cd pico_fm_transmitter
git clone https://github.com/raspberrypi/pico-sdk.git
cd pico-sdk
git submodule update --init
export PICO_SDK_PATH=`pwd`
cd ..
git clone https://github.com/kaduhi/pico-extras.git
cd pico-extras
git submodule update --init
export PICO_EXTRAS_PATH=`pwd`
cd ..
git clone https://github.com/kaduhi/pico-playground.git
cd pico-playground
mkdir build
cd build
cmake ..
make
cd apps/usb_sound_card
ls -l
```
You should see the usb_sound_card_fm_transmitter.uf2 file.
Just upload it to the Raspberry Pi Pico (RP2040) board.

## Possible future improvements

- Support lower clk_sys frequencies
  - I have tried on slower frequencies (e.g. 12MHz XO clock), but no USB support is a pain for debugging...
- Widen the frequency range
  - By toggling the two integer divider values not next to each other, e.g. 113 and 115
- Suppot the Raspberry Pi Pico2 board (RP2350)

#

Kazuhisa "Kazu." Terasaki

if you are insterested in, [here](https://www.instagram.com/kazuterasaki/) is my latest updates
