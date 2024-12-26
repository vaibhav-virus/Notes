### Understanding Timers in Arduino Uno

Timers in Arduino Uno are one of the most powerful tools to manage time-based tasks, such as blinking LEDs, controlling motors, or generating precise time delays. Arduino Uno is equipped with three timers:

- **Timer0**: 8-bit timer (used for functions like `millis()` and `delay()`).
- **Timer1**: 16-bit timer (useful for more precise timing).
- **Timer2**: 8-bit timer (commonly used for PWM and tone generation).

Each timer runs independently and is clocked from the system clock of 16 MHz in Arduino Uno.

---

### Key Concepts Related to Timers

1. **Prescalers**
   - A prescaler divides the clock frequency to slow down the timer. For instance, if the prescaler is set to 64, a 16 MHz clock becomes 16,000,000 / 64 = 250,000 Hz.

2. **Timer Modes**
   - Timers can operate in different modes, such as:
     - Normal Mode: Timer counts up until it overflows.
     - CTC Mode (Clear Timer on Compare): Timer resets when it reaches a predefined value.
     - PWM Mode: Used for generating Pulse Width Modulation signals.

3. **Interrupts**
   - Timers can trigger interrupts when they overflow or reach a compare match. These interrupts allow precise control over timing events.

4. **Registers**
   - Timers are controlled by specific registers in the microcontroller. For example:
     - `TCCRnA` and `TCCRnB`: Configure the timer mode and prescaler.
     - `TCNTn`: Holds the current timer value.
     - `OCRnA` and `OCRnB`: Define compare match values.
     - `TIMSKn`: Enables timer interrupts.
     - `TIFRn`: Flags indicating timer events.

---

### Example 1: Using Timer for LED Blinking (Normal Mode)

#### Objective:
Blink an LED every second using Timer1 in normal mode.

#### Code:
```cpp
volatile bool ledState = false;

void setup() {
  pinMode(13, OUTPUT); // Set pin 13 as output

  // Configure Timer1
  TCCR1A = 0;          // Normal mode
  TCCR1B = (1 << CS12) | (1 << CS10); // Prescaler of 1024

  TIMSK1 = (1 << TOIE1); // Enable overflow interrupt
  TCNT1 = 0;             // Initialize counter

  sei(); // Enable global interrupts
}

ISR(TIMER1_OVF_vect) {
  static uint16_t overflowCount = 0;
  overflowCount++;
  if (overflowCount >= 15) { // ~1 second at 1024 prescaler
    ledState = !ledState;
    digitalWrite(13, ledState);
    overflowCount = 0;
  }
}

void loop() {
  // Do nothing; all work is done in ISR
}
```

#### Explanation:
1. **Prescaler Calculation**:
   - Timer1 increments every 1024/16 MHz = 64 microseconds.
   - Timer1 overflows after 65536 counts (64 × 65536 = ~4.19 seconds).
   - To get a 1-second blink, count 15 overflows (~4.19 / 15 = ~1 second).

2. **Interrupt Handling**:
   - `ISR(TIMER1_OVF_vect)` is triggered on each overflow.
   - `overflowCount` keeps track of overflows to achieve a delay of approximately 1 second.

---

### Example 2: Generating a Square Wave Using CTC Mode

#### Objective:
Generate a 1 kHz square wave on pin 9 using Timer1 in CTC mode.

#### Code:
```cpp
void setup() {
  pinMode(9, OUTPUT); // Pin 9 as output

  // Configure Timer1 in CTC mode
  TCCR1A = (1 << COM1A0); // Toggle OC1A (pin 9) on compare match
  TCCR1B = (1 << WGM12) | (1 << CS10); // CTC mode, no prescaler

  OCR1A = 7999; // Compare match value for 1 kHz
}

void loop() {
  // Nothing needed in loop
}
```

#### Explanation:
1. **Frequency Calculation**:
   - No prescaler: Timer increments every 1/16 MHz = 62.5 ns.
   - For a 1 kHz square wave (500 Hz high + 500 Hz low):
     - Timer counts 16,000,000 / (2 × 1000) = 8000.
     - Subtract 1 for `OCR1A` value: 7999.

2. **Pin Toggle**:
   - The `COM1A0` bit in `TCCR1A` toggles pin 9 on every compare match.

---

### Example 3: PWM with Timer2

#### Objective:
Dim an LED using PWM on pin 3.

#### Code:
```cpp
void setup() {
  pinMode(3, OUTPUT); // Set pin 3 as output

  // Configure Timer2 for Fast PWM
  TCCR2A = (1 << WGM21) | (1 << WGM20) | (1 << COM2B1); // Fast PWM, non-inverting
  TCCR2B = (1 << CS22); // Prescaler of 64

  OCR2B = 128; // 50% duty cycle
}

void loop() {
  // Adjust brightness over time
  for (int i = 0; i < 256; i++) {
    OCR2B = i;  // Change duty cycle
    delay(10); // Smooth transition
  }
}
```

#### Explanation:
1. **PWM Mode**:
   - `WGM21` and `WGM20` enable Fast PWM mode.
   - `COM2B1` sets non-inverting mode, so pin 3 generates PWM.

2. **Duty Cycle**:
   - `OCR2B` sets the duty cycle: 128 = 50% (LED half-bright).

3. **Brightness Control**:
   - Loop adjusts `OCR2B` to vary brightness.

---

### Best Practices and Tips
1. **Use Hardware Timers When Precision Matters**:
   - For highly accurate time intervals, prefer hardware timers over `delay()` or `millis()`.

2. **Understand Timer Conflicts**:
   - Some Arduino functions (e.g., `millis()`, `delay()`) rely on Timer0. Avoid reconfiguring Timer0 unless necessary.

3. **Enable Global Interrupts (`sei()`)**:
   - When using interrupts, ensure global interrupts are enabled.

4. **Use Timer Libraries**:
   - For complex applications, libraries like `TimerOne` simplify timer usage.

### Using all timer interrupts
```cpp
static uint8_t b_count = 0;    // Counter for Compare Match B
static uint8_t overflow_count = 0; // Counter for Timer Overflow
// Compare Match A interrupt service routine
ISR(TIMER0_COMPA_vect) {
    PORTB ^= (1 << PB0); // Toggle LED1 (PB0) every 100ms
}

// Compare Match B interrupt service routine
ISR(TIMER0_COMPB_vect) {
    b_count++;
    if (b_count >= 2) { // 2 matches (200ms) for approximately 250ms
        PORTB ^= (1 << PB1); // Toggle LED2 (PB1)
        b_count = 0;         // Reset counter
    }
}

// Timer Overflow interrupt service routine
ISR(TIMER0_OVF_vect) {
    overflow_count++;
    if (overflow_count >= 62) { // Approximately 16ms * 62 = 1 second
        PORTB ^= (1 << PB2); // Toggle LED3 (PB2)
        overflow_count = 0;  // Reset counter
    }
}

void setup(void) {
    // Configure PB0, PB1, and PB2 as output (for LEDs)
    DDRB |= (1 << PB0) | (1 << PB1) | (1 << PB2);

    // Configure Timer0 in CTC mode
    TCCR0A = (1 << WGM01);               // Enable CTC mode (WGM01 = 1, WGM00 = 0)
    TCCR0B = (1 << CS01) | (1 << CS00);  // Set prescaler to 64

    // Set Compare Match values
    OCR0A = 249; // Compare Match A for 100ms
    OCR0B = 249; // Compare Match B for 100ms (to be counted for 250ms)

    // Enable interrupts for all Timer0 types
    TIMSK0 = (1 << OCIE0A) | (1 << OCIE0B) | (1 << TOIE0); // Enable Compare A, Compare B, and Overflow interrupts

    sei(); // Enable global interrupts
}

void loop(){}
```
