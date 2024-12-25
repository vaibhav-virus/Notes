***Bitwise registers SET, CLEAR and TOGGLE operations***

| **Operation**                | **Syntax**                                 | **Description**                              |
|------------------------------|--------------------------------------------|----------------------------------------------|
| **Creating a bit mask**      | `mask = ((1 << bit_position1) \| (1 << bit_position2);`         | Creates bit mask with specified bits set to `1`.
| **Set a bit**                | `register \|= (1 << bit_position);`         | Sets the specified bit to `1`.              |
| **Reset a bit**              | `register &= ~(1 << bit_position);`        | Clears the specified bit to `0`.            |
| **Toggle a bit**             | `register ^= (1 << bit_position);`         | Flips the specified bit's state.            |
| **Check if set**             | `if (register & (1 << bit_position))`      | Checks if the specified bit is `1`.         |
| **Check if cleared**         | `if (!(register & (1 << bit_position)))`   | Checks if the specified bit is `0`.         |
| **Set multiple bits**        | `register \|= ((1 << pos1) \| (1 << pos2));` | Sets multiple bits to `1`.                  |
| **Clear multiple bits**      | `register &= ~((1 << pos1) \| (1 << pos2));`| Clears multiple bits to `0`.                |
| **Toggle multiple bits**     | `register ^= ((1 << pos1) \| (1 << pos2));` | Toggles the state of multiple bits.         |
| **Check if multiple bits set**| `if ((register & mask) == mask)`          | Verifies all bits in a `mask` are `1`.      |
| **Set/clear/toggle with mask**| `register (Set)\|= (Clear)&=~ (Toggle)^= (mask);`  | Modifies bits defined in a `mask`.          |

***Register types and their use case.***

| **Register Type**            | **Common Names**           | **Use Case**                                       | **Significance**                                                                 |
|------------------------------|----------------------------|--------------------------------------------------|---------------------------------------------------------------------------------|
| **Data Direction Register**  | `DDRx` (e.g., `DDRB`)      | Configures pins as input (`0`) or output (`1`).   | Controls whether a pin is used for input or output, crucial for interfacing.   |
| **Port Data Register**       | `PORTx` (e.g., `PORTB`)    | Outputs logic levels to pins set as outputs.     | Used to drive digital signals to peripherals or LEDs.                          |
| **Pin Input Register**       | `PINx` (e.g., `PINB`)      | Reads the current state of input pins.           | Used for sensing input states from switches, sensors, etc.                     |
| **Interrupt Control Register**| `EIMSK`, `EIFR`, `PCMSKx` | Configures and handles external interrupts.      | Enables responsive actions to external signals (e.g., button press).           |
| **Timer/Counter Registers**  | `TCCRn`, `TCNTn`, `OCRn`   | Configures, tracks, and compares timer values.   | Used for precise timing, PWM generation, and event counting.                   |
| **Analog-to-Digital Converter (ADC) Registers**| `ADCSRA`, `ADMUX`      | Configures and controls the ADC module.          | Enables analog signal sampling and conversion to digital values.               |
| **USART (Serial) Registers** | `UCSRx`, `UDR`, `UBRRx`    | Configures and operates serial communication.    | Allows data exchange with other devices via UART protocol.                     |
| **Status Register**          | `SREG`                    | Tracks global status flags like carry, zero, etc.| Reflects the state of the processor after operations, crucial for decision-making.|
| **Watchdog Timer Register**  | `WDTCSR`                  | Configures the watchdog timer for resets.        | Prevents system lockup by resetting the microcontroller after a timeout.       |
| **Power and Sleep Registers**| `SMCR`, `PRR`             | Configures power-saving modes.                   | Reduces power consumption during inactive states.                              |
| **EEPROM Registers**         | `EEAR`, `EEDR`, `EECR`    | Handles EEPROM read/write operations.            | Allows non-volatile data storage.                                              |
| **SP (Stack Pointer) Register**| `SPH`, `SPL`             | Points to the current stack location.            | Manages function calls and local variables in memory.                          |
| **Interrupt Vector Table**   | Memory Mapped             | Maps interrupt service routines (ISRs).          | Defines the address of routines for handling interrupts.                       |
| **I/O Control Registers**    | `TWCR`, `SPCR`, `ACSR`    | Configures peripherals like I2C, SPI, and ADC.   | Allows communication with and control of external peripherals.                 |
| **Configuration Registers**  | Varies by MCU             | Sets up clock, oscillator, and fuse settings.    | Defines core microcontroller behavior, such as clock speed and reset behavior. |
