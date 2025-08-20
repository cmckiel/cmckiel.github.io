---
layout: post
title: "I2C: Finding the Right Abstraction"
---
# I2C: Finding the Right Abstraction
![](/assets/imu_gyro_data.gif)
> Gyroscope data coming from the MPU 6050 being printed out over UART. Logic analyzer capture of the I2C transaction in background. Development of this barebones MPU driver contributed significantly to the design of the I2C driver.

I think often, the difficulties in our code mirror the difficulties in our thinking. Good code often springs from thinking about the problem in the right way and begins by recognizing the true elements and entities at play. Using an abstraction ultimately says, "I hold these properties relevant and applicable to the problem I am solving and any other properties I hold irrelevant." It is a decision one makes, to consider some things and not others, often with no more justification than a hypothesis.
## Hypotheses Must Be Tested
Ultimately, as programmers, we must make what we think up. It must work functionally and aesthetically with all the things we have thought up and made up in the project previously. We must test our hypothesis against the world and confront the question, "Does this work the way I think it does?" And this is one of the many joys of programming. Engagement in the activity of programming is a constant refinement of your own understanding of the problem until your understanding is so deep and complete that you're able to write down explicit instructions even a machine could follow. A machine with no understanding or intuition of its own. Of course, our understanding is never quite complete and is the root cause of our inability to completely remove bugs from our code.

There are some who would establish design prior to development. The difficulty is, however, that your understanding of the problem is never worse than it is at the beginning. You haven't tested *any* of your initial thoughts. The trick with programming is that development *is* the design. The product *is* the specification. And the most effective path to quality software is to engage in the iterative process of incremental understanding by actually building what you've thought up and being prepared to completely change it.
## When Your First Idea Hits the Road, and it's Wrong
The first abstraction I had in mind for the I2C driver was the byte, a carryover from my UART development. I imagined communicating over I2C was nothing more than sending and receiving bytes. This led to an interface like this:
```C
HalStatus_t hal_i2c_init(void *config);

HalStatus_t hal_i2c_write(uint8_t device_addr, const uint8_t *data, size_t len, size_t *bytes_written, uint32_t timeout_ms);

HalStatus_t hal_i2c_read(uint8_t device_addr, uint8_t *data, size_t len, size_t *bytes_read, uint32_t timeout_ms);

HalStatus_t hal_i2c_write_read(uint8_t device_addr, const uint8_t *write_data, size_t write_len, uint8_t *read_data, size_t read_len, size_t *bytes_read, uint32_t timeout_ms);
```
At first glance it seems fine. Want to send data to a device? Use write() and feed it all your bytes. Want data from a device? Call read and have the data dropped into your buffer. This approach would work fairly well if the client was willing to wait and poll hardware for the results, keeping itself locked up until the entirety of the request is finished. But this wasn't an option for me. This HAL has plans to be used in a flight control system and the main loop needs to be able to compute the controls solution at definitive intervals, regardless of the I2C transmissions. Only by attempting to build out this interface did I realize the problem. (And how little I knew about I2C).

The next abstraction I tried out was a message. The interface stayed the same, but on the backend I took all of the parameters and formed an "I2C message". The messages were queued up so the ISR could work on each one autonomously while the main loop did its thing. This solved my first problem now that the main loop was free but created another. How do I manage returning results back to the client? My code completely failed for the common I2C case of write-read, where I first write to the device the register address I would like to read and then begin a second message that actually reads it. Say a client used write() to write the desired register address to the device. How shall a client know how to begin reading? Even if that was handled by the ISR, how does the client know which message the data it just received belonged to? If a client sent 10 messages, and gets back some data some time later, how should it know which message to associate with it? Keeping track of the order won't work, because what if something went wrong with the first 3 messages, and the client really received data that belongs with the 4th? Further, to nail one more nail in the coffin, what happens when there are multiple clients, all adding their own messages to the queue and all checking the results queue? When the driver gets results for a given message, who shall it send the results to? Should we implement a mailbox system with client UUIDs? No. This was clearly getting out of hand. The code was blowing up and I was no closer to the feature I really wanted.

## The Right Abstraction
Each of the attempts above were wrong but also necessary. Each taught me progressively more about I2C and the problem that I was solving. Each allowed me to test a hypothesis and get feedback. Eventually, I began to see I2C was not about sending bytes. The fundamental unit was not bytes or messages, but transactions, a series of bi-directional exchanges meant to accomplish a singular goal. Both parties had to actively participate perhaps many times during a single transaction. If I wanted my ISR to work quickly and silently in the background it needed a description of the entire transaction it was to perform, including the expected outcome. The second breakthrough was asking clients to maintain handles to their transactions. This way, clients can wait until their transaction is marked `COMPLETE` and then open it up to see the results. The driver never has to worry about who to send what. It simply works through the transactions it has been given, one at a time. This led to the following, much different, interface:
```C
typedef struct {
    // Immutable input. Can not change once transaction has been submitted.
    uint8_t target_addr;                     // The I2C address of the target device.
    HalI2C_Op_t i2c_op;                      // The type of transaction. i.e. READ, WRITE, or WRITE-READ.
    uint8_t tx_data[TX_MESSAGE_MAX_LENGTH];  // The data to send. Put the register addr in the first slot.
    size_t num_of_bytes_to_tx;               // The num of bytes to send the device. Include the reg addr in the count.
    size_t expected_bytes_to_rx;             // The desired number of bytes to read from the device during this transaction. Only set for READ or WRITE-READ.

    // Poll to determine when transaction has been completed.
    HalI2C_TxnState_t processing_state;      // Submit transaction with CREATED. When processing_state == COMPLETED then client can collect results. Safe to check periodically.

    // Results of the transaction. Only valid once processing_state == COMPLETED.
    // Initialize to prescribed values when submitting transaction.
    HalI2C_TxnResult_t transaction_result;   // Only valid once processing_state == COMPLETED. Contains the result of the transaction (success, fail, etc). Init to NONE.
    size_t actual_bytes_received;            // The actual number of bytes that got read during the transaction. Init to 0.
    size_t actual_bytes_transmitted;         // The actual number of bytes that got transmitted during the transaction. Init to 0.
    uint8_t rx_data[RX_MESSAGE_MAX_LENGTH];  // Data read from device will be stored here. Only valid after processing_state == COMPLETED.
                                             // Init rx_data to zeros when creating the transaction struct.
} HalI2C_Txn_t;

HalStatus_t hal_i2c_init(void *config);

HalStatus_t hal_i2c_submit_transaction(HalI2C_Txn_t *txn);

HalStatus_t hal_i2c_transaction_servicer();
```

## A Worthy Goal
All of these attempts were guided by the desire to accomplish a single goal. A benchmark of understanding that deeply excited me. I wanted to see live gyroscope data in my computer terminal. (As shown in the top of post). I wanted to shake my IMU and see the data go bananas as it tracks the angular velocities of my swings, rocks, and rolls. This single achievement would validate everything I have built thus far. The pursuit of this goal is what informed the design of my I2C driver. By actually attempting to build something on top and by actually *being* the client I discovered everything I disliked about the initial implementation. (And quickly). And build something, I did. I accomplished my goal and am grateful for the opportunity I had to learn more about I2C by *doing*.

![Gyroscope Data Path](/assets/gyro_data_path.jpg)
> The path of the gyroscope data as it moves from the MPU over I2C to the STM32F4, and from there over UART to the Raspberry Pi where the data can be viewed on the terminal by reading `/dev/ttyUSB0` using a tool like `screen`.
