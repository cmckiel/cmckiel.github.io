---
layout: post
title: "UART: Debugging a Critical Section"
categories: [HAL]
tags: [uart]
---
# UART: Debugging a Critical Section

I wrote this post because I *thought* I had written a great UART Driver. Referencing the data sheet, I developed the driver to spec on desktop with a test driven approach. The driver had 94% line coverage. Knowing the importance of hardware testing as well, I used the driver to echo a handful of ~30 byte messages. All lights were green: 48 passing tests, build and test workflows in CI, and on-target integration testing. I was eager to begin work on the next milestone. I almost did.
## Introduction

If you've been following some of my other work, you know the question I have been asking; Can we build embedded systems at the speed modern software allows while maintaining the rigor physical consequences demand? Building this driver was the latest in my attempts at understanding and answering that question. But so far, this driver was turning out to be much more rigor than speed. That's okay. Sometimes understanding the line means spending time on both sides.

A large amount of the enjoyment when building something is adding new capability — something that works that didn't exist before. Less of the enjoyment comes from testing the Nth corner case for something that already worked last week. But this was the path I had committed to and this was the work. And I had a nagging feeling something was missing. The on-target testing felt light. I needed a stress test.

Still unsure if the stress test I identified would be rigorous theater or genuine value add, I began development. I constructed a comprehensive message consisting of a wide range of characters and embedded data formats that was approximately two thousand bytes long. I would modify the test orchestrator to include this message and also later use this same message in a multi-hour soak test. Success meant the test orchestrator would send the message in a long and sustained burst to HAL (configured in loopback mode) and receive the very same bytes, in the same order, with no dropped bytes, and within a given timeframe. I expected it to pass. I was wrong.

On each test iteration, substantial amounts of data were missing. Checking cables didn't fix it. The driver consistently dropped data under sustained loads — loads comparable to potential applications on a real system. I couldn't help but smile — my stress test had caught something.

This post delves into what went wrong as well as how I narrowed down and eventually solved the issue, getting into some of the tools and techniques along the way. It also takes time to reflect on how the decision to pursue further testing resulted in finding and fixing a quality issue that would have significantly impacted downstream applications.

![Data Loss with Pretty Diff](/assets/UART_DataLoss_2ms_DualToggle_ShortDiff.png)
> UART loopback test diff. Characters in red were not echoed back to the test runner. This driver fails the loopback test.

## Background

A quick overview of the system architecture will help establish the context within which this post takes place. This critical insight here: TX and RX share an ISR.

![Uart Functional Diagram](/assets/UART_Architecture.drawio.svg)
> System diagram of HAL and STM32F4 with Pi Test Harness. Describes the data flow during loopback testing. HAL owns all software components represented on left of dotted line. STM32F4 chip owns the hardware registers on right side of dotted line inside its UART peripheral.

To describe the diagram above in words: when powered, the micro executes its main loop continually, checking for incoming data, echoing back whatever it receives, and performing a blocking delay. When the Pi test harness is prompted, it executes the python test script which connects to the serial port and begins sending a short message (32 bytes) followed by a long message (1911 bytes). The Pi then waits a short amount of time before reading the returning data and comparing sent vs received and issuing a test PASS/FAIL.
## Divide and Conquer

I knew, empirically, bytes were being dropped. But I didn't know what entity was responsible and why. I initially included the Pi Test Harness as part of the overall system and equally capable of producing this bug. After all, who tests the test? Isolating the problem meant first proving HAL was the cause and not the test harness. Establishing this was as simple as hooking up a logic analyzer to the RX and TX lines. The entire message could be observed during transmission from Pi to HAL and no bytes were missing. When measuring the return transmission from HAL to Pi, the bytes the test claimed missing were indeed missing. HAL was the source of this bug.

Moving again to divide the system in half, I asked if HAL's TX path or RX path was the cause. After pausing for a moment to discover a way to test this, I decided on a clever solution. I would bake the long test message into HAL and program the main loop to exclusively and repeatedly emit that message. This would remove any HAL receive bugs from the equation and focus only on transmit. I attached my analyzer to the line and confirmed all bytes were accounted for. It appeared that HAL's TX path was functioning well. I concluded the receive path was to blame. But, a question for the audience: When exactly can a system be divided into parts and reasoned about individually? And a question I should have asked myself: To what extent are the receive and transmit paths coupled?
## Irreducible Pieces

The RX path became the prime suspect. As such, it was subjected to many attempts to reveal its wrongdoing. But just like TX, when isolated, I saw no problem. The TX and RX paths worked fine on their own but not in a dynamic system together. This was clearly a problem relating to the interaction of components, not the components themselves. To answer the question I posed to the audience: parts of a system can only be reasoned about individually when the coupling between them is low or zero. In my case, the TX and RX paths were coupled via their shared ISR.

I focused the investigation on this point of interaction. In particular, I looked at the critical sections responsible for protecting the shared circular buffers. Accessing the shared buffers from the application loop required temporarily disabling interrupts. This protection was necessary, but was it a problem? I needed more information. How long were interrupts being disabled? How often? And critically, are they disabled long enough to cause the amount of data loss I was measuring?
## Instrumentation

I needed to know how long the ISR was being disabled. I had two options. The first is a common practice in embedded. The idea is to flip a GPIO pin whenever some event you care about happens in code. That way external observers can measure the exact timings of those events. In my case this meant toggling a GPIO high when entering the critical section and toggling it back to low on exit. A logic analyzer can then record the time spent high and correlate it to other events it is measuring such as incoming UART data. The second method is to use an ARM debugging feature called Data Watchpoint and Trace (DWT) to count the exact CPU cycles spent inside the critical section. In order to read that cycle count, live on-target debugging needs to be set up. I decided to do both.
### GPIO + Logic Analyzer

![UART Data Loss GPIO Logic Analyzer Capture](/assets/UART_DataLoss_2ms_DualToggle_Short.png)
> Logic Analyzer Capture to isolate root cause of data loss. Signal D2 in red represents bytes coming in over UART to HAL (RX), signal D1 in brown are the bytes HAL echoes back to sender (TX). D3 in yellow is a GPIO pin being toggled around HAL's TX critical section. 661 µs is a long time to have ISRs disabled. Enough time for ~7 bytes to come in at 115200 baud.

After inserting the GPIO toggle statements around the TX critical section, I ran the test and collected the above result. 661 µs is a long time for interrupts to be disabled, and this is very much a problem. The capture also paints a clear picture that describes the mechanics of the system. First, data begins arriving on the RX pin while HAL is doing its 2 ms delay. After 2ms, HAL wakes up and reads the bytes it has received so far. It immediately deposits those bytes into the TX circular buffer, disabling interrupts to do so. After interrupts are reenabled (GPIO pin goes low) bytes can be observed leaving the board as the ISR picks one byte at a time off the TX circular buffer and deposits into the transmit data register (DR).

This also explains why short messages pass the loopback test, while the long message revealed the issue. A short message has a high likelihood of being entirely received during HAL's sleep window where that isn't possible for a long message. Bytes of a long message are still incoming when the HAL wakes up and begins a transmit that prevents it from receiving. Playing with the delay provided interesting results. For example, increasing the loop delay significantly to 500 ms resulted in no data loss due to transmit attempts, but *did* result in data loss due to circular buffer wraparound. (RX buffer cannot hold the entire message)

One non-solution immediately presented itself. I could fit any message size by simply making the delay longer and buffer larger. But multi-second latency was no real solution. I was convinced the critical section would need work but I was still curious if the second method of measuring the critical section would confirm the same result.
### DWT + On-Target Debug

This stage was about collecting additional supporting evidence as well as finally setting up on-target debugging — something I had wanted to do for a while now. What follows is a quick rundown of the setup.

You can initialize DWT with the following code:
```c
CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;   // enable trace
DWT->CYCCNT = 0;                                  // reset counter
DWT->CTRL  |= DWT_CTRL_CYCCNTENA_Msk;             // start counting
```

and you can count the cycles in a given section takes like so:
```c
uint32_t start = DWT->CYCCNT;

// Code under test

uint32_t cycles = DWT->CYCCNT - start; // cycles: the number of cycles the code under test took to execute
```

Physical setup for live debugging:
![Remote Debug Diagram](/assets/RemoteDebugDiagram.svg)
> Diagram showing data flow and tooling required for live debugging setup.

The idea was to count the number of cycles each pass of the critical section and keep track of the maximum number of cycles observed, stored in a variable `max_cycles`. Below is a VS Code screenshot during a live debug session:

![UART Data Loss Live Debug](/assets/UART_DataLoss_2ms_LiveDebug.png)
> Screenshot of live debugging. On left in the `watch` panel, `max_cycles = 11534` records the highest water mark observed during the test script's execution. At 16 MHz base clock frequency, this is a ~721 µs delay. Consistent with randomly observed ~661 µs delay on logic analyzer capture.

Using this method I was able to show once again that the critical section was far too long. It was also a great opportunity to set up live debugging for my platform.
## The Smoking Gun

The smoking gun is here: a zoomed in frame from the logic capture focusing on "over the lazy dog" next to test output showing the missing bytes.

![Side by Side Data Loss](/assets/UART_SideBySide_DataLoss.drawio.png)
> Logic capture and the test output side by side. Red signal represents incoming data. Yellow signal high represents ISR disabled, low represents enabled. We can see that 'e' is received and moved into DR by hardware while the transmit side becomes active, disabling interrupts. ' ', 'l', 'a', 'z' are all squarely lost as they overwrite each other in the shift register. 'e' stays in DR the whole time. 'y' is the value in the shift register right when interrupts are re-enabled, 'e' gets read from DR, 'y' gets shifted to DR, and the rest of the stream continues processing.

## The Root Cause and The Design Tradeoff

Below is the transmit critical section for one of the UART channels. The critical section appears quite long in hindsight. I was aware of general advice, "keep your critical sections and ISRs short". But how short is *short*? The answer of course is relative to the application. Testing the way I did above helped me understand the exact time scales that mattered for my application. I also acquired the meta-skill of knowing how to characterize *short* for other applications.

When I originally wrote this critical section, I was making a specific design choice. I wanted to make atomicity guarantees to clients. That is why I entered the critical section first, ensured there would be enough space to fit the entire message, and only then began writing bytes to the buffer. To make this guarantee, I couldn't exit the critical section until the entire message was written to the buffer. Otherwise, in a multi-client scenario, a second client could start a write that fills up the reserved space, resulting in a partial message being written to the buffer.

Making these guarantees required the critical section to be so long that the driver lost data. This was clearly the wrong trade to make.

Code under test including instrumentation:
```c
HalStatus_t stm32f4_uart1_write(const uint8_t *data, size_t len)
{
	HalStatus_t res = HAL_STATUS_ERROR;
	size_t current_buffer_capacity = 0;

	if (data && uart1_initialized)
	{
		hal_gpio_toggle_led();           // Toggle the GPIO HIGH as we enter the critical section. (Visible on Logic Analyzer capture)
		uint32_t start = DWT->CYCCNT;    // Store the cycle start count to later compute the high watermark.
		CRITICAL_SECTION_ENTER();

		bool buffer_was_empty = circular_buffer_is_empty(&tx_ctx);

		if (circular_buffer_get_current_capacity(&tx_ctx, &current_buffer_capacity) &&
			0 < len && len <= current_buffer_capacity)
		{
			res = HAL_STATUS_OK;
			// The entirety of the data will fit.
			// Put the data into buffer.
			for (size_t i = 0; i < len; i++)
			{
				if (!circular_buffer_push_no_overwrite(&tx_ctx, data[i]))
				{
					// Something went wrong. We were unable to push despite
					// there being capacity.
					res = HAL_STATUS_ERROR;
					break;
				}
			}

			if (res == HAL_STATUS_OK && buffer_was_empty)
			{
				USART1->CR1 |= USART_CR1_TXEIE;  // Enable TXE interrupt
			}

		}

		CRITICAL_SECTION_EXIT();                                 // (ISR can fire immediately, prior to GPIO toggle and cycle count - causing slight inaccuracy in measurements)
		hal_gpio_toggle_led();                                   // Bring the GPIO LOW as we exit
		last_cycles = DWT->CYCCNT - start;                       // Compute the number of cycles
		if (last_cycles > max_cycles) max_cycles = last_cycles;  // Remember the high water mark
	}

	return res;
}
```

Critical sections disabled ISRs system wide for each channel, each transmit, and each receive. This is way too heavy handed.
```c
#define CRITICAL_SECTION_ENTER() __disable_irq()
#define CRITICAL_SECTION_EXIT()  __enable_irq()
```
## The Solution

No loops. No nested function calls. Different client guarantees. Channel-based ISR masking. Much faster.

What allowed this new and better design was first, in a multi-client scenario, clients must use appropriate mutual exclusion techniques to ensure this UART channel is properly shared. The second is that clients are no longer guaranteed that their entire message will fit in the buffer before attempting a write. There is enough information for clients to determine when this happens so they can decide to resend or not. But occasionally this could result in partial messages being sent out over the line. Another important change was to remove the heavy-handed global interrupt disabling in favor of per-channel ISR masking. Previously, UART1 and UART2 were each disabling interrupts system wide on each transmit.

This new tradeoff significantly improved performance. The cost is that clients may occasionally write partial messages. Clients must also manage the resource sharing overhead of the channel. For my system, this was the right call. Many of the applications over UART will have higher level protocols with CRC and other mechanisms for catching partial messages.

Revised TX function:
```c
hal_status_t stm32f4_uart1_write(const uint8_t *data, size_t len, size_t *bytes_written)
{
	hal_status_t res = HAL_STATUS_ERROR;
	bool push_success = true;

	if (uart1_initialized && bytes_written && data && len > 0)
	{
		*bytes_written = 0;
		for (size_t i = 0; i < len; i++)
		{
			// ***************************************************
			// CRITICAL SECTION ENTER
			NVIC_DisableIRQ(USART1_IRQn);
			push_success = circular_buffer_push_no_overwrite(&tx_ctx, data[i]);
			NVIC_EnableIRQ(USART1_IRQn);
			// CRITICAL SECTION EXIT
			// ***************************************************

			if (push_success)
			{
				*bytes_written += 1;
			}
			else
			{
				break;
			}
		}

		// If bytes were written successfully to buffer, then enable the transmit
		// interrupt because those bytes need to be sent out.
		if (*bytes_written > 0)
		{
			USART1->CR1 |= USART_CR1_TXEIE;  // Enable TXE interrupt
		}

		// If we successfully wrote all bytes to buffer, then the function was an
		// overall success.
		res = (*bytes_written == len) ? HAL_STATUS_OK : HAL_STATUS_ERROR;
	}

	return res;
}
```

Rerunning the same tests with the new version of the driver showed the measurable performance benefits I was looking for. The highest observed cycle count for the critical section was 1096 cycles, a ~10x speedup. Further, this number works on the timescales of the UART driver. 1096 cycles is about 68.5 µs @ 16 MHz or ~8 bits received at 115200 baud. Each UART frame (8N1) is 10 bits. So, ISRs are never disabled long enough to miss a single byte.

To increase my confidence levels further, I set up the stress test to run repeatedly with a `watch` command and let the test run for 5 hours. Zero bytes were lost over the course of this test.

![High Water Mark](/assets/UART_DataLoss_Fixed_LiveDebug.png)
![UART Data Loss Fix Logic Capture](/assets/UART_DataLoss_Fixed_2ms.png)
> Observed measurements show 44 µs ISR disabled time. 5 bits can be received during this time at 115200 baud. UART frames are 10 bits for my configuration so this fits comfortably. Importantly, UART2 ISR is not disabled during this 44 µs UART1 TX window. This fix made each UART channel responsible for masking only its own interrupts.

## Closing Thoughts

The stress test I almost skipped caught a real bug. Skipping it would have looked like faster progress — more demonstrable features in the same window, happier managers, the works. But the driver wasn't fit for applications, and pushing that defect downstream to debug alongside application code, with other development quirks hiding around it, would have been harder, not easier. Sometimes going fast means going slow. I'm glad I took the time here.

Technically, the biggest takeaway for me was learning how to instrument a real-time system and reason about the true time constraints for my system. I believe this skill will transfer with me as I continue in this field. Particularly in flight control software, where control loop jitter, actuator latency budgets, and real-time determinism play important and persistent roles.

On the rigor vs speed question, one observation stands out: the integration stress test was easily the highest-value test I ran, because it was the closest to real system operation. Tests that approximate reality are inherently more valuable. If "moving fast" means maximizing value created per unit time, then on any horizon longer than a few weeks, the test that catches the application-breaking bug _is_ the fast path. The slow path is the one that feels fast today and becomes weight on your shoulders tomorrow.
## Open Questions

What was the value of my unit tests? I don't have a clean answer, but I have a suspicion. The unit tests got me to roughly 90% correct before I ever flashed the chip — buffer overflow behavior, control sequence bit patterns, data sheet conformance, the kind of edge cases that are painful to provoke from the integration level. When I did bring it up on hardware, it essentially worked. That foundation is what made the timing bug _findable_; if the buffer logic had also been broken, I'd have been chasing two bugs at once through a much narrower window.

What I'm less sure about is the ratio. Forty-eight tests and 94% coverage is a lot of artifact for code that still shipped with a critical-section bug invisible to any of them. Would I write the same suite again, or write half of it and spend the saved time on integration stress tests earlier? I lean toward the latter, but I'm not certain. The honest version of the rigor-vs-speed question isn't "tests yes or no" — it's where on the unit-to-integration spectrum the marginal hour pays off most.
