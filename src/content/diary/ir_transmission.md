---
title: "IR Transmission"
date: 2026-10-16
summary: "Encoding 3 unique values into a single byte of data to be transmitted over IR Signal"
---

For my ence260 assignment, we need to make two  ATmega32u2 microcontroller's communicate through a infrared LED and a 36 kHz reciever. To achieve this, we set the game to have 7 different states which the controller is able to switch between after receiving a event message. These event messages are encoded as ASCII characters so they can be sent in a single byte, removing the chance for a partial packet to be lost if we had sent it over multiple bytes.

The boards implmented **ir_uart**, a university provided IR sending and recieving foundation. This has functions alter the messages sent by the board such as...

- **ir_uart_putc :** Writes a character to IR_UART. This function checks if data has been recieved and if so reads the data throws it away based on the assumption that is electrical or optical echoing. Afterwards sends the message out from the infrared light.
- **ir_uart_getc :** Attempts to read a character from IR_UART, but is blocked if nothing is available to read.

Using these two functions, we were able to send signals between the two controllers with limited interference from other infrared light. We used this infrared to transmit details about the game state and the ball as it moves from one screen to another.  The IR also needed to know the state of the other controller, such as if it connected, if it missed the ball, if it acknowledged thre previous IR message.

```c
#define IR_START 'S'
#define IR_ACK 'A'
#define IR_GO 'G'
#define IR_LOST 'L'
```

But this is only for the basics communication and doesnt transfer as much information as whe a ball croses over to the other side. When we transfer information about the ball over we send over a uint8_t containing the balls location, direction and velocity. However these are unable to be easily sent in one byte so we encode the data and send through a signal to be decoded by the reciever. This encoding means all information about the ball is sent in one byte, preventing the loss of information.
```c

/* Ball encoding equation needs to be larger than the ascii values previously mentioned so they dont accidently overlap */

void comms_send_ball (uint8_t row, int8_t row_dir, uint8_t power)
{
uint8_t dir_index = (uint8_t) (row_dir - AIM_DIAG_LEFT);
uint8_t code = (power * AIM_COUNT + dir_index) * BOARD_ROWS + row;
ir_uart_putc (IR_BALL_BASE + code);
}

/* ball decoding
the ball has to carry row(0...6), row_dir(-1,0,1) and power(0...6)
code = (power * AIM_COUNT + dir_index) * BOARD_ROWS + row
*/
if (byte >= IR_BALL_BASE && byte < IR_BALL_BASE + IR_BALL_CODES)
{
	uint8_t code = byte - IR_BALL_BASE;
	
	msg->type = COMMS_BALL;
	msg->row = code % BOARD_ROWS;
	code /= BOARD_ROWS;
	msg->row_dir = (int8_t) (code % AIM_COUNT) + AIM_DIAG_LEFT;
	msg->power = code / AIM_COUNT;
	return true;
}
```

As the ball needs to carry three individual pieces of information, we need to find a number that is able to contain all that possible information. There are 7 different rows the ball could be in, 3 different directions  (left, straight, right) and 7 different options for speed (So that would be 7 *  3 * 7 total different combinations). However, we do **not** want this to overlap with the previously stated ASCII Characters so we have to increase by at least the lowest value ('S' = 84.)  However if we had use a character with a value higher than 109 ('m'), then we would go over the total size of a byte (255.)

To combine each of the numbers, we... (To be updated..)