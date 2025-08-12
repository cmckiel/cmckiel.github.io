---
layout: post
title: "I2C First Signs of Life"
---

# I2C First Signs of Life

![Oscilloscop readout of I2C address](/assets/i2c_first_address_oscilloscope.png)
> I2C frame. SDA in blue. SCL in yellow. Address of peripheral is 0x42.

As part of my hal from scratch project (which is a subcomponent of my drone from scratch project) I am implementing an I2C driver, well, from... scratch 🙃. My philosophy is to get feedback as fast as possible and this bare-bones implementation lets me do just that.

## An Initial Implementation

Polling based, no read, write only but only writes an address... not exactly production grade. But wow, was I happy to see that initial frame on the oscope. The main challenge I overcame with this was learning about all the peripheral initialization and how to get it to actually send something. Even once it sent something, it took me a minute to correctly interpret what I was seeing. Setting my oscilloscope triggers specifically targeted at I2C was helpful, and I'd like to share that with you too. In this post I want to use Controller/Target rather than Master/Slave. One of the things I want to share in this post is how to initialize the STM32F4 I2C peripheral for Controller mode.

## Initializing the I2C Peripheral

There are two main steps to initializing a polling based I2C driver:

- Set up the GPIO pins for I2C
- Set up the I2C peripheral itself

### GPIO First
```
static void configure_gpio()
{
    // Enable Bus.
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOBEN;

    // Set PB8 (i2c1 SCL pin) mode to alternate function.
    GPIOB->MODER &= ~BIT_16;
    GPIOB->MODER |= BIT_17;

    // Set PB9 (i2c1 SDA pin) mode to alternate function.
    GPIOB->MODER &= ~BIT_18;
    GPIOB->MODER |= BIT_19;

    // Set PB8 alternate function type to I2C (AF04)
    GPIOB->AFR[1] &= ~(0xF << (PIN_0 * AF_SHIFT_WIDTH));
    GPIOB->AFR[1] |= (AF4_MASK << (PIN_0 * AF_SHIFT_WIDTH));

    // Set PB9 alternate function type to I2C (AF04)
    GPIOB->AFR[1] &= ~(0xF << (PIN_1 * AF_SHIFT_WIDTH));
    GPIOB->AFR[1] |= (AF4_MASK << (PIN_1 * AF_SHIFT_WIDTH));

    // Open drain
    GPIOB->OTYPER |= (GPIO_OTYPER_OT_8 | GPIO_OTYPER_OT_9);
}
```

The main things that have to happen:
- Choose your pins. I chose PB8 and PB9 because they were available and right next to each other on my dev board.
- Once the pins are chosen, the bus clock needs to be enabled for the gpio's port. In my case, port B. Without this clock, the peripheral is "dead" and it consumes near zero power. Providing clock brings it to life. It's like watering a plant. 🌱
- Gpio pins can support a variety of functions. They have a setting that lets them know they're being used for an Alternative Function. This must be set for both pins.
- The next question the gpio asks it's developer is "What function? I2C? Uart? Something else?" Devs must answer. In my case, I2C is alternative function 4.
- The gpio pins must be set to open drain. Essentially, this is electrically necessary based on the I2C protocol. The bus lines are always supposed to be tied high. The peripheral communicates over the lines by pulling to ground. If any of the actors on the bus are attempting to actively drive the line high, they could get shorted.

If you happen to be wondering how someone would know to do most of these things, I wanted to let you know that it is all in the user manual, data sheet, and reference manual for the board. But a little bit of experience helps too. Someday I might make a course on this. Comment if you'd like something like that.

### Peripheral Second

```
static void configure_peripheral()
{
    // Send the clock to I2C1
    RCC->APB1ENR |= RCC_APB1ENR_I2C1EN;

    // Need to set the APB1 Clock frequency in the CR2 register.
    // With no dividers, it is the same as the System Frequency of 16 MHz.
    I2C1->CR2 &= ~(I2C_CR2_FREQ);
    I2C1->CR2 |= (SYS_FREQ_MHZ & I2C_CR2_FREQ);

    // Time to rise (TRISE) register. Set to 17 via the calcualtion
    // Assumed 1000 ns SCL clock rise time (maximum permitted for I2C Standard Mode)
    // Periphal's clock period (1 / SYSTEM_FREQ_MHZ)
    // (1000ns / 62.5) = 17 OR SYSTEM_FREQ_MHZ + 1 = 17. Either calc works.
    size_t trise_reg_val = SYS_FREQ_MHZ + 1;
    I2C1->TRISE &= ~(I2C_TRISE_TRISE);
    I2C1->TRISE |= (trise_reg_val & I2C_TRISE_TRISE);

    // Set CCR.
    // We want to setup CCR so that the peripheral can count up ticks of the
    // peripheral bus clock, and use that count to create the SCL clock at 100kHz for
    // Standard Mode.
    // 100 kHz SCL means 1 / 100kHz period of 10 microseconds.
    // So we need to transition the SCL clock every ~5 microseconds.
    // On a 16 MHz bus clock with a tick every 62.5 nanoseconds, this means
    // we need to transition the SCL line every 80 ticks to achieve 100kHz SCL line.
    size_t ticks_between_scl_transitions = 80;
    I2C1->CCR &= ~(I2C_CCR_CCR);
    I2C1->CCR |= (ticks_between_scl_transitions & I2C_CCR_CCR);

    // Standard mode
    I2C1->CCR &= ~I2C_CCR_FS;

    // Enable the peripheral.
    I2C1->CR1 |= I2C_CR1_PE;
}
```

Here are the main steps to initializing the I2C Peripheral:

- Water your peripheral by feeding it clock. 💧🌱⚡
- Let your peripheral know what kind of clock you're feeding it. (16 MHz)
- Follow the reference manual's prescribed calculation to compute the TRISE (Time to Rise) register's value. Essentially, the maximum time an SCL clock can take to rise in standard mode according to the I2C protocol is 1000 ns. Assume the worst, and put the calculated value in the register.
- Put the right 🪄magic number in the CCR register. The concept is actually simple, since the hardware was designed to be simple (and cheaper). This is essentially what you're telling the peripheral: use the peripheral bus clock to count. Count to X and toggle the SCL line. But what should X be? We want to operate our I2C on Standard mode, which is a 100kHz SCL. So how many clock cycles of 16 MHz do I need to count to alternate the SCL line to achieve 100 kHz clock? Put that number in CCR.
- Ensure the peripheral is operating in Standard Mode.
- Enable the peripheral. It's cleared for action.

With these steps now completed, it should be possible to use your I2C peripheral to send information on the bus!

## Using the I2C Peripheral

```
HalStatus_t hal_i2c_write(uint8_t slave_addr, const uint8_t *data, size_t len, size_t *bytes_written, uint32_t timeout_ms)
{
    if (bytes_written) {
        *bytes_written = 0;
    }

    // Send START condition
    I2C1->CR1 |= I2C_CR1_START;

    // Wait for START condition to be generated (SB flag)
    while (!(I2C1->SR1 & I2C_SR1_SB)) {
        // Could add timeout here
    }

    // Send slave address + write bit (0)
    I2C1->DR = (slave_addr << 1) | 0;

    // Wait for address to be sent (ADDR flag) or acknowledge failure (AF flag)
    while (!(I2C1->SR1 & (I2C_SR1_ADDR | I2C_SR1_AF))) {
        // Could add timeout here
    }

    if (I2C1->SR1 & I2C_SR1_AF) {
        // Acknowledge failure - slave didn't respond
        I2C1->SR1 &= ~I2C_SR1_AF; // Clear AF flag
        I2C1->CR1 |= I2C_CR1_STOP; // Send STOP
        i2c_stats.tx_errors++;
        return HAL_STATUS_ERROR;
    }

    // Clear ADDR flag by reading SR1 then SR2
    (void)I2C1->SR1;
    (void)I2C1->SR2;

    // Send STOP condition (for now, just testing address transmission)
    I2C1->CR1 |= I2C_CR1_STOP;

    return HAL_STATUS_OK;
}
```

```
int main(void)
{
	size_t bytes_written_i2c = 0;
	hal_i2c_init(NULL);

	while (1)
	{
		bytes_written_i2c = 0;
		hal_i2c_write(0x42, NULL, 0, &bytes_written_i2c, 0);
		hal_delay_ms(20);
	}

	return 0;
}
```

Read the code or not. It is just there for reference. But what's there is enough to get our wheels spinning. The full setup is depicted below, to include the 5.1K pull-up resistors to tie the lines high. I used that resistance value because I had it easily on hand. Anything from 2K-10K is common. There is a lot to consider when optimizing these values, and this isn't the post to get into it.

![Microcontroller and Oscope Measuring I2C Bus](../assets/i2c_initial_testing.jpg)
> STM32F4 Dev Board in white. Raspberry Pi testing harness. Oscilloscope reads the I2C Bus.

There it is. The first signs of life. This driver has hope. And it is far from done.
