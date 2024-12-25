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
