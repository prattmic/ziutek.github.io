---
layout: post
title: Go on very small hardware (Part 1)
tags: mcu go emgo
---

How low we can *Go* and still do something useful?

I recently bought this ridiculously cheap board:

![STM32F030F4P6]({{ site.baseur }}/images/mcu/f030-demo-board/board.jpg)

I bought it for three reasons. First, I have never dealt (as a programmer) with STM32F0 series. Second, the STM32F10x series is getting old. MCUs belonging to the STM32F0 family are just as cheap if not cheaper and has newer peripherals, with many improvements and bugs fixed. Thirdly, I chose the smallest member of the family for the purpose of this article, to make the whole thing a little more intriguing.

<!--more-->

## The Hardware

The [STM32F030F4P6](http://www.st.com/content/st_com/en/products/microcontrollers/stm32-32-bit-arm-cortex-mcus/stm32-mainstream-mcus/stm32f0-series/stm32f0x0-value-line/stm32f030f4.html) is impresive piece of hardware:

* CPU: [Cortex M0](https://en.wikipedia.org/wiki/ARM_Cortex-M#Cortex-M0) 48 MHz (only 12000 logic gates, in minimal configuration),
* RAM: 4 KB,
* Flash: 16 KB,
* ADC, SPI, I2C, USART and a couple of timers,

all enclosed in TSSOP20 package. As you can see, it is very small 32-bit system.

## The software

If you hoped to see how to use [genuine Go](https://golang.org/) to program this board, you need to read the hardware specification one more time. You must face the truth: there is a negligible chance that someone will ever add support for Cortex-M0 to the Go compiler and this is just the beginning of work.

I'll use [Emgo](https://github.com/ziutek/emgo), but don't worry, you will see that it gives you as much Go as it can on such small system.

There was no support for any F0 MCU in [stm32/hal](https://github.com/ziutek/emgo/tree/master/egpath/src/stm32/hal) before this board arrived to me. After brief study of [RM](http://www.st.com/resource/en/reference_manual/dm00091010.pdf), the STM32F0 series appeared to be striped down STM32F3 series, which made work on new port easier.

If you want to follow subsequent steps of this post, you need to install Emgo

```
cd $HOME
git clone https://github.com/ziutek/emgo/
cd emgo/egc
go install
```

and set a couple environment variables

```
export EGCC=path_to_arm_gcc      # eg. /usr/local/arm/bin/arm-none-eabi-gcc
export EGLD=path_to_arm_linekr   # eg. /usr/local/arm/bin/arm-none-eabi-ld
export EGAR=path_to_arm_archiver # eg. /usr/local/arm/bin/arm-none-eabi-ar

export EGROOT=$HOME/emgo/egroot
export EGPATH=$HOME/emgo/egpath

export EGARCH=cortexm0
export EGOS=noos
export EGTARGET=f030x6
```

A more detailed description can be found on the [Emgo website](https://github.com/ziutek/emgo).

Ensure that egc is on your PATH. You can use `go build` instead of `go install` and copy egc to your *$HOME/bin* or */usr/local/bin*.

Now create new directory for your first Emgo program and copy example linker script there:
```
mkdir $HOME/firstemgo
cd $HOME/firstemgo
cp $EGPATH/src/stm32/examples/f030-demo-board/blinky/script.ld .
```

## Minimal program

Lets create minimal program in *main.go* file:

```go
package main

func main() {
}
```

It's actually minimal and compiles witout any problem:

```
$ egc
$ arm-none-eabi-size cortexm0.elf
   text    data     bss     dec     hex filename
   7452     172     104    7728    1e30 cortexm0.elf
```

The first compilation can take some time. The resulting binary takes 7624 bytes of Flash (text+data), quite a lot for a program that does nothing. There are 8760 free bytes left to do something useful.

What about traditional *Hello, World!* code:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
```

Unfortunately, this time it went worse:

```
$ egc
/usr/local/arm/bin/arm-none-eabi-ld: /home/michal/P/go/src/github.com/ziutek/emgo/egpath/src/stm32/examples/f030-demo-board/blog/cortexm0.elf section `.text' will not fit in region `Flash'
/usr/local/arm/bin/arm-none-eabi-ld: region `Flash' overflowed by 10880 bytes
exit status 1
```

*Hello, World!* requires at last STM32F030x6, with its 32 KB of Flash.

The *fmt* package forces to include whole *strconv* and *reflect* packages. All three are pretty big, even a slimmed-down versions in Emgo. We must forget about it. There are many applications that don't require fancy formatted text output. Often one or more LEDs or seven segment display are enough. However, in Part 2, I'll try to use *strconv* package to format and print some numbers and text over UART. 

## Blinky

Our board has one LED connected between PA4 pin and VCC. This time we need a bit more code:

```go
package main

import (
	"delay"

	"stm32/hal/gpio"
	"stm32/hal/system"
	"stm32/hal/system/timer/systick"
)

var led gpio.Pin

func init() {
	system.SetupPLL(8, 1, 48/8)
	systick.Setup(2e6)

	gpio.A.EnableClock(false)
	led = gpio.A.Pin(4)

	cfg := &gpio.Config{Mode: gpio.Out, Driver: gpio.OpenDrain}
	led.Setup(cfg)
}

func main() {
	for {
		led.Clear()
		delay.Millisec(100)
		led.Set()
		delay.Millisec(900)
	}
}
```

By convention, the *init* function is used to initialize the basic things and configure peripherals.

`system.SetupPLL(8, 1, 48/8)` configures RCC to use PLL with external 8 MHz oscilator as system clock source. PLL divider is set to 1, multipler to 48/8 = 6 which gives 48 MHz system clock.

`systick.Setup(2e6)` setups Cortex-M SYSTICK timer as system timer, which runs the scheduler every 2e6 nanoseconds (500 times per second).

`gpio.A.EnableClock(false)` enables clock for GPIO port A. *False* means that this clock should be disabled in low-power mode, but this is not implemented int STM32F0 series.

`led.Setup(cfg)` setups PA4 pin as open-drain output.

`led.Clear()` sets PA4 pin low, which in open-drain configuration turns the LED on.

`led.Set()` sets PA4 to high-impedance state, which turns the LED off.

Lets compile this code:

```
$ egc
$ arm-none-eabi-size cortexm0.elf
   text    data     bss     dec     hex filename
   9772     172     168   10112    2780 cortexm0.elf
```

As you can see, blinky takes 2320 bytes more than minimal program. There are still 6440 bytes left for more code.

Let's see if it works:

```
$ openocd -d0 -f interface/stlink.cfg -f target/stm32f0x.cfg -c 'init; program cortexm0.elf; reset run; exit'
Open On-Chip Debugger 0.10.0+dev-00319-g8f1f912a (2018-03-07-19:20)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
debug_level: 0
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
none separate
adapter speed: 950 kHz
target halted due to debug-request, current mode: Thread 
xPSR: 0xc1000000 pc: 0x0800119c msp: 0x20000da0
adapter speed: 4000 kHz
** Programming Started **
auto erase enabled
target halted due to breakpoint, current mode: Thread 
xPSR: 0x61000000 pc: 0x2000003a msp: 0x20000da0
wrote 10240 bytes from file cortexm0.elf in 0.817425s (12.234 KiB/s)
** Programming Finished **
adapter speed: 950 kHz
```

For this article, the first time in my life, I converted short video to [animated PNG](https://en.wikipedia.org/wiki/APNG) sequence. I'm impressed, oodbye YouTube and sorry IE users. See [apngasm](http://apngasm.sourceforge.net/) for more info. I should study HTML5 based alternative, but for now, APNG is my preffered way for short looped videos.

![STM32F030F4P6]({{ site.baseur }}/images/mcu/f030-demo-board/blinky.png)

## More Go

If you aren't a Go programmer but you've heard something about Go language, you can say: "This syntax is nice, but not a significant improvement over C. Show me *Go language*, give mi *channels* and *gorutines!*".

Here you are:

```go
import (
	"delay"

	"stm32/hal/gpio"
	"stm32/hal/system"
	"stm32/hal/system/timer/systick"
)

var led1, led2 gpio.Pin

func init() {
	system.SetupPLL(8, 1, 48/8)
	systick.Setup(2e6)

	gpio.A.EnableClock(false)
	led1 = gpio.A.Pin(4)
	led2 = gpio.A.Pin(5)

	cfg := &gpio.Config{Mode: gpio.Out, Driver: gpio.OpenDrain}
	led1.Setup(cfg)
	led2.Setup(cfg)
}

func blinky(led gpio.Pin, period int) {
	for {
		led.Clear()
		delay.Millisec(100)
		led.Set()
		delay.Millisec(period - 100)
	}
}

func main() {
	go blinky(led1, 500)
	blinky(led2, 1000)
}
```

Code changes are minor: the second LED was added and the previous *main* function was renamed to *blinky* and now requires two parameters. *Main* starts first *blinky* in new gorutine, so both LEDs are handled *concurrently*. It is worth mentioning that *gpio.Pin* type supports concurrent access to different pins of the same GPIO port.

Emgo still has several shortcomings. One of them is that you have to specify a maximum number of gorutines (tasks) in advance. It's time to edit *script.ld*:

```
ISRStack = 1024;
MainStack = 1024;
TaskStack = 1024;
MaxTasks = 2;

INCLUDE stm32/f030x4
INCLUDE stm32/loadflash
INCLUDE noos-cortexm
```

The size of the stacks are set by guess, and we'll not care about them at the moment.

```
$ egc
$ arm-none-eabi-size cortexm0.elf
   text    data     bss     dec     hex filename
  10020     172     172   10364    287c cortexm0.elf
```

Another LED and gorutine costs 248 bytes of Flash.

![STM32F030F4P6]({{ site.baseur }}/images/mcu/f030-demo-board/gorutines.png)

## Channels

Channels are the [preffered way](https://blog.golang.org/share-memory-by-communicating) in Go to communicate between gorutines. Emgo goes even further and allows to use *buffered* channels by *interrupt handlers*. The next example actually shows such case.

```go
package main

import (
	"delay"
	"rtos"

	"stm32/hal/gpio"
	"stm32/hal/irq"
	"stm32/hal/system"
	"stm32/hal/system/timer/systick"
	"stm32/hal/tim"
)

var (
	leds  [3]gpio.Pin
	timer *tim.Periph
	ch    = make(chan int, 1)
)

func init() {
	system.SetupPLL(8, 1, 48/8)
	systick.Setup(2e6)

	gpio.A.EnableClock(false)
	leds[0] = gpio.A.Pin(4)
	leds[1] = gpio.A.Pin(5)
	leds[2] = gpio.A.Pin(9)

	cfg := &gpio.Config{Mode: gpio.Out, Driver: gpio.OpenDrain}
	for _, led := range leds {
		led.Set()
		led.Setup(cfg)
	}

	timer = tim.TIM3
	pclk := timer.Bus().Clock()
	if pclk < system.AHB.Clock() {
		pclk *= 2
	}
	freq := uint(1e3) // Hz
	timer.EnableClock(true)
	timer.PSC.Store(tim.PSC(pclk / freq))
	timer.ARR.Store(700) // ms
	timer.DIER.Store(tim.UIE)
	timer.CR1.Store(tim.CEN)

	rtos.IRQ(irq.TIM3).Enable()
}

func blinky(led gpio.Pin, period int) {
	for range ch {
		led.Clear()
		delay.Millisec(100)
		led.Set()
		delay.Millisec(period - 100)
	}
}

func main() {
	go blinky(leds[1], 500)
	blinky(leds[2], 500)
}

func timerISR() {
	timer.SR.Store(0)
	leds[0].Set()
	select {
	case ch <- 0:
		// Success
	default:
		leds[0].Clear()
	}
}

//c:__attribute__((section(".ISRs")))
var ISRs = [...]func(){
	irq.TIM3: timerISR,
}
```

Changes compared to the previous example:

1. Thrid LED was added and connected to PA9 pin (TXD pin on UART header). 

2. The timer (TIM3) has been introduced as a source of interrupts.

3. The new *timerISR* function handles *irq.TIM3* interrupt.

4. The new buffered channel with capacity 1 is intended for communication between *timerISR* and *blinky* gorutines. 

5. The *ISRs* array acts as *interrupt vector table*, a parto of bigger *exception vector table*.

6. The *blinky's for statement* was replaced with a *range statement*.

For convenience, all LEDs, or rather their pins, have been collected in the *leds* array. Additionally, all pins have been set to a known initial state (high), just before they were configured as outputs.

In this case, we want the timer to tick at 1 kHz. To configure TIM3 prescaler, we need to known its input clock frequency. According to RM the input clock frequency is equal to APBCLK when APBCLK = AHBCLK, otherwise it is equal to 2 x APBCLK.

If the CNT register is incremented at 1 KHz, then the value of ARR register corresponds to the period of counter *update event* (reload event) expressed in milliseconds. To make update event to generate interrupts, the UIE bit in DIER register must be set. The CEN bit enables the timer.

Timer peripheral should stay enabled in low-power mode, to keep ticking when the CPU is put to sleep: `timer.EnableClock(true)`. It doesn't matter in case of STM32F0 but it's important for code portability.

The *timerISR* function handles *irq.TIM3* interrupt requests. `timer.SR.Store(0)` clears all event flags in SR register to deassert the IRQ to [NVIC](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0432c/Cihbecee.html). The rule of thumb is to clear the interrupt flags immedaitely at begining of their handler, because of the IRQ deassert latency. This prevents unjustified re-call the handler again. For absolute certainty, the clear-read sequence should be performed, but in our case, just clearing is enough.

The following code:

```go
select {
case ch <- 0:
	// Success
default:
	leds[0].Clear()
}
```

is a Go way to non-blocking sending on a channel. No one interrupt handler can afford to wait for a free space in the channel. If the channel is full, the default case is taken, and the onboard LED is set on, until the next interrupt.

The *ISRs* array contains interrupt vectors. The `//c:__attribute__((section(".ISRs")))` causes that the linker will inserted it into .ISRs section.

The new form of *blinky's for* loop:

```go
for range ch {
	led.Clear()
	delay.Millisec(100)
	led.Set()
	delay.Millisec(period - 100)
}
```

is the equivalent of:

```go
for {
	_, ok := <-ch
	if !ok {
		break // Channel closed.
	}
	led.Clear()
	delay.Millisec(100)
	led.Set()
	delay.Millisec(period - 100)
}
```

Note that in this case we aren't interested in the value received from the channel. We're interested only in the fact that there is something to receive. We can give it expression by declaring the channel's element type as empty struct `struct{}` instead of *int* and send `struct{}{}` values instead of 0, but it can be strange for newcomer's eyes.

Lets compile this code:

```
$ egc
$ arm-none-eabi-size cortexm0.elf
   text    data     bss     dec     hex filename
  11096     228     188   11512    2cf8 cortexm0.elf
```

This new example takes 11324 bytes of Flash, 1132 bytes more than the previous one.

With the current timings, both *blinky* gorutines consume from the channel much faster than the *timerISR* sends to it. So they both wait for new data simultaneously and you can observe the randomness of *select*, required by the [Go specification](https://golang.org/ref/spec#Select_statements).

![STM32F030F4P6]({{ site.baseur }}/images/mcu/f030-demo-board/channels1.png)

The onboard LED is always off, so the channel overrun never occurs.

Let's speed up sending, by changing `timer.ARR.Store(200)` to `timer.ARR.Store(700)`. Now the *timerISR* sends 5 messages per second but both recipients together can receive only 4 messages per second.

![STM32F030F4P6]({{ site.baseur }}/images/mcu/f030-demo-board/channels2.png)

As you can see, the *timerISR* lights the yellow LED which means there is no space in the channel.

This is where I finish the first part of this article. You should know that this part didn't show you the most important thing in Go language, *interfaces*.

Gorutines and channels are only nice and convenient syntax. You can replace them with your own code - not easy but feasible. Interfaces are the essence of Go,
and that's what I will start with in the next part of this article.

We still have some free space on Flash.