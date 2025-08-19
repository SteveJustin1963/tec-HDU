# tec-HDU 

tec1 to ATA HDU 


# Read-in: what this design and code do

Statement: this is an 8-bit, I/O-mapped ATA/IDE interface for a Z80. It exposes the ATA registers on Z80 ports \$40–\$47 and uses a separate port \$48 to latch the high byte so 16-bit IDE words can be moved over an 8-bit Z80 bus. Transfers are one sector (512 bytes = 256 words) at a time, with polling of ATA status (BSY, DRDY, DRQ, ERR) to pace reads/writes. &#x20;

Key hardware model, in plain terms:

* 74HC245 is the data transceiver; DIR follows read/write so data flows the right way; OE gates the bus. The Z80 sees IDE data at port \$40; the “other byte” is held in a 74HC574 latch read/written at \$48.
* Two 74HC139 decoders and some glue (OR/AND) derive /CS0, /IORD, /IOWR from Z80 /RD, /WR, A0–A3, clock, and the selected I/O address window (the “PORT SELECT”). The timing charts show Z80 port cycles lining up with ATA t1–t4 windows.
* A small RC and LED handle /DMACK, /RST, /DASP as per ATA basics.

What the Z80 code provides:

* Ports: ide\_register0..7 = \$40..\$47 (data and task-file regs), ide\_high\_byte = \$48 for the latch.&#x20;
* Entry points: ide\_get\_id, ide\_read\_sector, ide\_write\_sector. Each returns Carry=1 on success, or A=error (A=0 means timeout).&#x20;
* LBA setup: writes sector count=1 (reg2), LBA0..LBA2 (regs3..5), and high nibble + LBA mode + drive bit into reg6 (0xE0 | drive<<4 | LBA\[24..27]).&#x20;
* Polling:

  * wait\_busy\_ready: loops until BSY=0 and DRDY=1 (status bits 7:6 = 01). First access allows \~5s for spin-up, then \~1s thereafter (tracked in ide\_status bits).&#x20;
  * wait\_buffer: loops until DRQ=1 (status bit 3).&#x20;
  * test\_error: checks ERR (bit0) or WF (bit5); if set, reads error register (reg1) and clears Carry.&#x20;
* Data moves (note the “big-endian word” trick so ASCII from IDENTIFY looks right):

  * Read: for 256 words, IN low byte from \$40 then IN high byte from \$48; store as high,low in RAM.&#x20;
  * Write: for 256 words, OUT high byte to \$48 (pre-load latch) then OUT low byte to \$40 (commits the full word to IDE).&#x20;

Definition folded in: the interface is a latch-assisted, 8-bit PIO bridge that maps ATA task-file registers to Z80 I/O space and splits each 16-bit data word into two 8-bit operations while honoring ATA status/handshake.

Two small examples (evidence):

1. Identify device (master)

* Write reg6 = 0xA0 (LBA mode + master).
* Poll until BSY=0, DRDY=1.
* OUT reg7 = 0xEC.
* Poll BSY/DRDY, check ERR, then wait DRQ=1 and read 256 words via \$40/\$48 into sector\_buffer.
  This exactly matches ide\_get\_id and the shared read path.&#x20;

2. Read LBA=0x00000001 (slave)

* Write reg2=1, reg3=0x01, reg4=0x00, reg5=0x00, reg6=(0xE0|0x10|0)=0xF0 (LBA, slave, high nibble=0).
* Poll BSY/DRDY; OUT reg7=0x20 (READ SECTOR).
* Poll BSY/DRDY, check ERR; wait DRQ=1; read 256 words via \$40/\$48.
  These steps are ide\_setup\_lba + ide\_read\_sector flow.&#x20;

# MINT IDE Interface for TEC-1

Looking at your Z80 assembly IDE code and the MINT manual, I can help you convert this to run on the TEC-1. This will be a significant conversion since we're going from low-level assembly to MINT's higher-level operations.I've converted your Z80 assembly IDE interface code to MINT for the TEC-1. Here are the key aspects of the conversion:

## Major Changes from Assembly to MINT:

1. **Port I/O**: Uses MINT's `/I` (input) and `/O` (output) operators instead of Z80 `IN`/`OUT` instructions
2. **Buffer Management**: Uses MINT's heap allocation (`/A`) for the 512-byte sector buffer
3. **Variables**: Maps assembly variables to MINT's single-letter variables (a-z)
4. **Flow Control**: Converts assembly jumps/calls to MINT's loop and conditional structures
5. **Error Handling**: Simplified to boolean returns rather than complex flag checking

## Testing the Code:

Start with basic initialization and drive detection:

```mint
// Load the code into MINT, then:
I           // Initialize IDE system
G           // Try to get drive ID
```

If successful, try reading sector 0:
```mint
0 V         // Select master drive
0 0 0 0 A   // Set LBA to sector 0  
S           // Read sector
P           // Print contents if successful
```

## Hardware Considerations:

- Ensure your TEC-1 has the IDE interface hardware connected to ports $40-$48
- The timing diagrams you provided show the interface expects proper setup/hold times
- MINT's interpreted nature adds delays that may actually help with timing

## Potential Issues:

1. **Performance**: MINT is interpreted, so operations will be slower than assembly
2. **Timing**: Some IDE drives may be sensitive to command timing
3. **Buffer Access**: MINT's array operations are less efficient than direct memory access

## Optimizations:

If you encounter timing issues, you can add delays:
```mint
10()        // Add delay loops where needed
```






## Port Definitions and Variables Setup

```mint
// IDE register port addresses (same as assembly version)
// ide_register0 = $40, ide_register1 = $41, etc.
// ide_high_byte = $48

// Global variables for IDE operations
// a = ide_status (bit 0=drive select, bit 1=master accessed, bit 2=slave accessed)  
// b = sector_buffer array pointer
// c = ide_lba0 (LSB)
// d = ide_lba1  
// e = ide_lba2
// f = ide_lba3 (MSB)
// g = carry flag from operations
// h = remainder/overflow flag
// i = loop counter (built-in)
// j = outer loop counter (built-in)
// k = temp storage
// l = temp storage 2
// m = temp storage 3
// n = temp storage 4

// Initialize IDE system
:I
0 a !           // Clear IDE status
512 /A b !      // Allocate 512 byte sector buffer
0 c ! 0 d ! 0 e ! 0 f !  // Clear LBA registers
;
```

## Core Helper Functions

```mint
// Wait for drive to be ready (not busy)
:W
0 k !           // timeout counter low
0 l !           // timeout counter high
/U (            // unlimited loop
  // Check IDE status bit 
  a 1 & 1 + m !  // get bit 1 or 2 to test
  m { m !        // shift left to get bit position
  
  a m & 0 = (    // if first time access
    50 n !       // longer timeout for spin-up
  ) /E (
    10 n !       // shorter timeout
  )
  
  n ( )          // delay loop
  k 1 + k !      // increment counter
  k 0 = (        // if low counter wraps
    l 1 + l !    // increment high counter
    l 0 = (      // if both wrap (timeout)
      0           // return 0 (failure)
      /F /W       // exit loop
    )
  )
  
  71 /I          // read status from port $47 (ide_register7)
  " 192 &        // mask busy(7) and ready(6) bits  
  64 ^           // want busy=0, ready=1
  0 = (          // if condition met
    // Mark drive as accessed
    a m | a !    // set the accessed bit
    /T           // return true (success)
    /F /W        // exit loop
  )
  /T /W          // continue loop
)
;

// Test for IDE errors
:E
71 /I          // read status from register 7
" 1 & 0 = (    // if error bit not set
  /T           // return true (no error)
) /E (
  " 32 & 0 > ( // if write error bit set
    71 /I      // return status as error code
  ) /E (
    65 /I      // read error register ($41)
  )
  0 =          // return false if error
)
;

// Wait for data request (buffer ready)
:D
0 k ! 0 l !    // timeout counters
/U (           // unlimited loop
  50 ( )       // delay
  k 1 + k !    // increment counter
  k 0 = (
    l 1 + l !
    l 0 = (    // timeout
      0        // return failure
      /F /W
    )
  )
  71 /I        // read status
  8 &          // test DRQ bit (bit 3)
  0 > (        // if buffer ready
    /T         // return success
    /F /W
  )
  /T /W
)
;
```

## LBA Setup Function

```mint
// Setup LBA address for IDE operation
:L
1 66 /O        // set sector count to 1 (port $42)
c 67 /O        // LBA 0:7 to port $43
d 68 /O        // LBA 8:15 to port $44  
e 69 /O        // LBA 16:23 to port $45

// Setup LBA 24:27 + mode bits
f 15 &         // mask to 4 bits
224 |          // set bits 5:7 = 111 for LBA mode

// Add master/slave bit
a 1 & 0 > (    // if slave selected
  16 |         // set bit 4
)
70 /O          // write to port $46 (ide_register6)
;

// Master/slave select helper
:M
a 1 & 0 > (    // if bit 0 set (slave)
  " 16 |       // set bit 4 on stack value
)
;
```

## Buffer Read/Write Functions

```mint
// Read 512 bytes from IDE buffer to sector_buffer
:R
0 i !          // counter for 256 words
256 (          // loop 256 times
  64 /I        // read low byte from port $40
  k !          // store low byte
  72 /I        // read high byte from port $48 (latch)
  
  // Store as word in buffer (big-endian for ASCII compatibility)
  b i 2 * ?    // calculate buffer position
  " ! k !      // store high byte first
  b i 2 * 1 + ? // next position  
  k !          // store low byte
  
  i 1 + i !    // increment counter
)
;

// Write 512 bytes from sector_buffer to IDE
:Y
0 i !          // counter
256 (          // loop 256 times
  // Get word from buffer
  b i 2 * ? k ! // get high byte
  b i 2 * 1 + ? l ! // get low byte
  
  k 72 /O      // send high byte to latch (port $48)
  l 64 /O      // send low byte to port $40 (triggers write)
  
  i 1 + i !
)
;
```

## Main IDE Functions

```mint
// Read sector from IDE drive
:S
L              // setup LBA address
W 0 = (        // wait for ready, if failed
  0            // return 0 (failure)  
) /E (
  32 71 /O     // send read sector command ($20) to port $47
  W 0 = (      // wait for ready
    0          // return failure
  ) /E (
    E 0 = (    // test for errors  
      0        // return failure
    ) /E (
      D 0 = (  // wait for buffer ready
        0      // return failure
      ) /E (
        R      // read buffer
        /T     // return success
      )
    )
  )
)
;

// Write sector to IDE drive  
:T
L              // setup LBA
W 0 = (        // wait ready
  0
) /E (
  48 71 /O     // send write command ($30) to port $47
  W 0 = (
    0
  ) /E (
    E 0 = (
      0  
    ) /E (
      D 0 = (
        0
      ) /E (
        Y      // write buffer
        W 0 = (
          0
        ) /E (
          E    // final error check
        )
      )
    )
  )
)
;

// Get IDE drive identification
:G  
160 M          // setup drive select with mode bits
70 /O          // select device (port $46)
W 0 = (        // wait ready
  0
) /E (
  236 71 /O    // send identify command ($EC) to port $47
  W 0 = (
    0
  ) /E (
    E 0 = (
      0
    ) /E (  
      D 0 = (
        0
      ) /E (
        R        // read identification data
        /T       // return success
      )
    )
  )
)
;
```

## Utility Functions

```mint
// Set LBA address (pass 4 bytes on stack: lba3 lba2 lba1 lba0)
:A
f ! e ! d ! c !  // pop LBA bytes in reverse order
;

// Set drive select (0=master, 1=slave)  
:V
a 254 & |       // clear bit 0 and set new value
a !
;

// Print sector buffer contents (for debugging)
:P
`Sector data:` /N
0 i !
512 (
  b i ? .       // print byte value
  32 /C         // print space
  i 15 & 15 = ( // every 16 bytes
    /N          // new line
  )
  i 1 + i !
)
;

// Print LBA address
:X  
`LBA: ` f , e , d , c , /N  // print in hex
;
```

## Usage Examples

```mint
// Initialize IDE system
I

// Select master drive and read sector 0
0 V              // select master drive  
0 0 0 0 A       // set LBA to 0
S (             // if read successful
  `Read successful` /N
  P             // print sector contents
) /E (
  `Read failed` /N  
)

// Get drive identification
G (
  `Drive ID successful` /N
  P
) /E (
  `Drive ID failed` /N
)

// Write test pattern to sector 1
1 0 0 0 A       // set LBA to 1

// Fill buffer with test pattern
0 i !
512 (
  i 255 & b i ?!  // fill with 0-255 pattern
  i 1 + i !
)

T (             // write sector
  `Write successful` /N
) /E (
  `Write failed` /N
)
```

## Notes

1. **Port Addresses**: Uses same ports as original assembly ($40-$48)
2. **Error Handling**: Returns 0 for failure, non-zero for success
3. **Buffer Management**: 512-byte buffer allocated with `/A` operator
4. **Timeouts**: Implemented with nested counters for long delays
5. **Drive Selection**: Bit 0 of variable `a` selects master(0) or slave(1)
6. **LBA Mode**: Always uses LBA addressing mode (not CHS)

## Limitations

- No interrupt support (polling only)
- Simplified timeout handling compared to assembly version
- Error codes simplified to boolean success/failure
- Buffer access is byte-oriented rather than optimized word access


# added a comprehensive menu system to the MINT IDE interface. 
previous code - was just the core IDE functions. 

Here's what the menu provides:

## Main Menu Options:
1. **Initialize IDE System** - Sets up the interface
2. **Select Drive** - Choose master or slave drive  
3. **Get Drive Information** - Reads and displays drive ID, model, serial
4. **Read Sector** - Read any sector by LBA address
5. **Write Sector** - Write to any sector (with confirmation)
6. **Dump Sector** - Display sector contents in hex
7. **Fill Buffer** - Fill with test patterns (sequential, alternating, zeros, 0xFF)
8. **Show Current LBA** - Display current LBA address
9. **Set LBA Address** - Change the target sector address
0. **Exit** - Quit the program

## To Start the Menu System:

```mint
// After loading all the code:
O
```

This will display the main menu and handle all user interaction with simple number choices.

## Menu Features:

- **User-friendly prompts** for all inputs
- **Error checking** for invalid menu choices
- **Safety confirmations** for write operations  
- **Hex display** of sector data
- **Drive information extraction** showing model and serial number
- **Multiple test patterns** for buffer filling
- **LBA address management** with easy input

# MINT IDE Interface for TEC-1

## Port Definitions and Variables Setup

```mint
// IDE register port addresses (same as assembly version)
// ide_register0 = $40, ide_register1 = $41, etc.
// ide_high_byte = $48

// Global variables for IDE operations
// a = ide_status (bit 0=drive select, bit 1=master accessed, bit 2=slave accessed)  
// b = sector_buffer array pointer
// c = ide_lba0 (LSB)
// d = ide_lba1  
// e = ide_lba2
// f = ide_lba3 (MSB)
// g = carry flag from operations
// h = remainder/overflow flag
// i = loop counter (built-in)
// j = outer loop counter (built-in)
// k = temp storage
// l = temp storage 2
// m = temp storage 3
// n = temp storage 4

// Initialize IDE system
:I
0 a !           // Clear IDE status
512 /A b !      // Allocate 512 byte sector buffer
0 c ! 0 d ! 0 e ! 0 f !  // Clear LBA registers
;
```

## Core Helper Functions

```mint
// Wait for drive to be ready (not busy)
:W
0 k !           // timeout counter low
0 l !           // timeout counter high
/U (            // unlimited loop
  // Check IDE status bit 
  a 1 & 1 + m !  // get bit 1 or 2 to test
  m { m !        // shift left to get bit position
  
  a m & 0 = (    // if first time access
    50 n !       // longer timeout for spin-up
  ) /E (
    10 n !       // shorter timeout
  )
  
  n ( )          // delay loop
  k 1 + k !      // increment counter
  k 0 = (        // if low counter wraps
    l 1 + l !    // increment high counter
    l 0 = (      // if both wrap (timeout)
      0           // return 0 (failure)
      /F /W       // exit loop
    )
  )
  
  71 /I          // read status from port $47 (ide_register7)
  " 192 &        // mask busy(7) and ready(6) bits  
  64 ^           // want busy=0, ready=1
  0 = (          // if condition met
    // Mark drive as accessed
    a m | a !    // set the accessed bit
    /T           // return true (success)
    /F /W        // exit loop
  )
  /T /W          // continue loop
)
;

// Test for IDE errors
:E
71 /I          // read status from register 7
" 1 & 0 = (    // if error bit not set
  /T           // return true (no error)
) /E (
  " 32 & 0 > ( // if write error bit set
    71 /I      // return status as error code
  ) /E (
    65 /I      // read error register ($41)
  )
  0 =          // return false if error
)
;

// Wait for data request (buffer ready)
:D
0 k ! 0 l !    // timeout counters
/U (           // unlimited loop
  50 ( )       // delay
  k 1 + k !    // increment counter
  k 0 = (
    l 1 + l !
    l 0 = (    // timeout
      0        // return failure
      /F /W
    )
  )
  71 /I        // read status
  8 &          // test DRQ bit (bit 3)
  0 > (        // if buffer ready
    /T         // return success
    /F /W
  )
  /T /W
)
;
```

## LBA Setup Function

```mint
// Setup LBA address for IDE operation
:L
1 66 /O        // set sector count to 1 (port $42)
c 67 /O        // LBA 0:7 to port $43
d 68 /O        // LBA 8:15 to port $44  
e 69 /O        // LBA 16:23 to port $45

// Setup LBA 24:27 + mode bits
f 15 &         // mask to 4 bits
224 |          // set bits 5:7 = 111 for LBA mode

// Add master/slave bit
a 1 & 0 > (    // if slave selected
  16 |         // set bit 4
)
70 /O          // write to port $46 (ide_register6)
;

// Master/slave select helper
:M
a 1 & 0 > (    // if bit 0 set (slave)
  " 16 |       // set bit 4 on stack value
)
;
```

## Buffer Read/Write Functions

```mint
// Read 512 bytes from IDE buffer to sector_buffer
:R
0 i !          // counter for 256 words
256 (          // loop 256 times
  64 /I        // read low byte from port $40
  k !          // store low byte
  72 /I        // read high byte from port $48 (latch)
  
  // Store as word in buffer (big-endian for ASCII compatibility)
  b i 2 * ?    // calculate buffer position
  " ! k !      // store high byte first
  b i 2 * 1 + ? // next position  
  k !          // store low byte
  
  i 1 + i !    // increment counter
)
;

// Write 512 bytes from sector_buffer to IDE
:Y
0 i !          // counter
256 (          // loop 256 times
  // Get word from buffer
  b i 2 * ? k ! // get high byte
  b i 2 * 1 + ? l ! // get low byte
  
  k 72 /O      // send high byte to latch (port $48)
  l 64 /O      // send low byte to port $40 (triggers write)
  
  i 1 + i !
)
;
```

## Main IDE Functions

```mint
// Read sector from IDE drive
:S
L              // setup LBA address
W 0 = (        // wait for ready, if failed
  0            // return 0 (failure)  
) /E (
  32 71 /O     // send read sector command ($20) to port $47
  W 0 = (      // wait for ready
    0          // return failure
  ) /E (
    E 0 = (    // test for errors  
      0        // return failure
    ) /E (
      D 0 = (  // wait for buffer ready
        0      // return failure
      ) /E (
        R      // read buffer
        /T     // return success
      )
    )
  )
)
;

// Write sector to IDE drive  
:T
L              // setup LBA
W 0 = (        // wait ready
  0
) /E (
  48 71 /O     // send write command ($30) to port $47
  W 0 = (
    0
  ) /E (
    E 0 = (
      0  
    ) /E (
      D 0 = (
        0
      ) /E (
        Y      // write buffer
        W 0 = (
          0
        ) /E (
          E    // final error check
        )
      )
    )
  )
)
;

// Get IDE drive identification
:G  
160 M          // setup drive select with mode bits
70 /O          // select device (port $46)
W 0 = (        // wait ready
  0
) /E (
  236 71 /O    // send identify command ($EC) to port $47
  W 0 = (
    0
  ) /E (
    E 0 = (
      0
    ) /E (  
      D 0 = (
        0
      ) /E (
        R        // read identification data
        /T       // return success
      )
    )
  )
)
;
```

## Utility Functions

```mint
// Set LBA address (pass 4 bytes on stack: lba3 lba2 lba1 lba0)
:A
f ! e ! d ! c !  // pop LBA bytes in reverse order
;

// Set drive select (0=master, 1=slave)  
:V
a 254 & |       // clear bit 0 and set new value
a !
;

// Print sector buffer contents (for debugging)
:P
`Sector data:` /N
0 i !
512 (
  b i ? .       // print byte value
  32 /C         // print space
  i 15 & 15 = ( // every 16 bytes
    /N          // new line
  )
  i 1 + i !
)
;

// Print LBA address
:X  
`LBA: ` f , e , d , c , /N  // print in hex
;
```

## Menu System

```mint
// Main menu display
:Q
`=====================================` /N
`        TEC-1 IDE Interface          ` /N
`=====================================` /N
`1. Initialize IDE System` /N
`2. Select Drive (Master/Slave)` /N
`3. Get Drive Information` /N
`4. Read Sector` /N
`5. Write Sector` /N
`6. Dump Sector (Hex)` /N
`7. Fill Buffer with Pattern` /N
`8. Show Current LBA` /N
`9. Set LBA Address` /N
`0. Exit` /N
`=====================================` /N
`Enter choice: `
;

// Get user input for menu choice
:U
/K k !          // read keyboard input
k 48 - k !      // convert ASCII to number
k
;

// Main menu loop
:O
/U (            // unlimited loop
  Q             // display menu
  U m !         // get user choice
  /N /N
  
  m 1 = ( I `IDE System Initialized` /N /N )
  m 2 = ( C )   // drive selection
  m 3 = ( H )   // get drive info  
  m 4 = ( J )   // read sector
  m 5 = ( K )   // write sector
  m 6 = ( P )   // dump sector
  m 7 = ( B )   // fill buffer
  m 8 = ( X )   // show LBA
  m 9 = ( N )   // set LBA
  m 0 = ( 
    `Goodbye!` /N 
    /F /W       // exit loop
  )
  
  m 0 < m 9 > | ( // if invalid choice
    `Invalid choice! Press any key...` /N
    /K '        // wait for keypress and drop it
  )
  
  `Press any key to continue...` /N
  /K '          // wait and drop keypress
  /N /N
  /T /W         // continue loop
)
;

// Drive selection menu
:C
`Select Drive:` /N
`0 = Master Drive` /N  
`1 = Slave Drive` /N
`Enter choice: `
U n !           // get choice
n 0 = n 1 = | ( // if valid choice
  n V           // set drive
  `Drive ` n . ` selected` /N
) /E (
  `Invalid drive selection` /N
)
;

// Get drive information
:H
`Getting drive information...` /N
G (             // if successful
  `Drive ID Read Successfully` /N
  `Model: `
  // Extract model name from ID buffer (chars 54-93)
  27 i !        // start at word 27 (byte 54)
  20 (          // 20 words = 40 chars
    b i 2 * ? /C     // print high byte
    b i 2 * 1 + ? /C // print low byte  
    i 1 + i !
  )
  /N
  
  `Serial: `
  // Extract serial (chars 20-39) 
  10 i !
  10 (
    b i 2 * ? /C
    b i 2 * 1 + ? /C
    i 1 + i !
  )
  /N
) /E (
  `Drive ID Failed` /N
)
;

// Read sector menu
:J
`Enter LBA address (4 bytes, LSB first):` /N
`LBA0 (0-255): ` U c ! /N
`LBA1 (0-255): ` U d ! /N  
`LBA2 (0-255): ` U e ! /N
`LBA3 (0-255): ` U f ! /N

`Reading sector...` /N
S (             // if read successful
  `Sector read successfully` /N
  `First 64 bytes:` /N
  0 i !
  64 (
    b i ? k !
    k 16 < ( `0` )  // leading zero for hex
    k ,             // print in hex
    32 /C           // space
    i 15 & 15 = ( /N ) // newline every 16
    i 1 + i !
  )
) /E (
  `Sector read failed` /N
)
;

// Write sector menu  
:K
`WARNING: This will overwrite data!` /N
`Enter LBA address (4 bytes, LSB first):` /N
`LBA0 (0-255): ` U c ! /N
`LBA1 (0-255): ` U d ! /N
`LBA2 (0-255): ` U e ! /N  
`LBA3 (0-255): ` U f ! /N

`Are you sure? (1=Yes, 0=No): `
U n !
n 1 = (         // if confirmed
  `Writing sector...` /N
  T (           // if write successful
    `Sector written successfully` /N
  ) /E (
    `Sector write failed` /N
  )
) /E (
  `Write cancelled` /N
)
;

// Fill buffer with test pattern
:B
`Select pattern:` /N
`1. Sequential (0,1,2,3...)` /N
`2. Alternating (0xAA,0x55...)` /N
`3. All zeros` /N
`4. All ones (0xFF)` /N
`Enter choice: `
U n !

n 1 = (         // sequential
  0 i !
  512 (
    i 255 & b i ?!
    i 1 + i !
  )
  `Buffer filled with sequential pattern` /N
)

n 2 = (         // alternating
  0 i !
  512 (
    i 1 & ( 85 ) /E ( 170 ) // 0x55 or 0xAA
    b i ?!
    i 1 + i !
  )
  `Buffer filled with alternating pattern` /N
)

n 3 = (         // zeros
  0 i !
  512 (
    0 b i ?!
    i 1 + i !
  )
  `Buffer filled with zeros` /N
)

n 4 = (         // ones
  0 i !
  512 (
    255 b i ?!
    i 1 + i !
  )
  `Buffer filled with 0xFF` /N
)

n 1 < n 4 > | (
  `Invalid pattern selection` /N
)
;

// Set LBA address menu
:N
`Current LBA: ` c . d . e . f . /N
`Enter new LBA (4 bytes):` /N
`LBA0 (LSB): ` U c ! /N
`LBA1:      ` U d ! /N
`LBA2:      ` U e ! /N  
`LBA3 (MSB):` U f ! /N
`LBA set to: ` c . d . e . f . /N
;
```

## Quick Start

```mint
// Load all the IDE functions above, then start the menu:
O

// This will display the main menu and handle all user interaction
```

## Usage Examples

```mint
// Simple command-line usage (without menu):

// Initialize and test
I               // Initialize IDE
0 V             // Select master drive
G               // Get drive info

// Read sector 0  
0 0 0 0 A       // Set LBA to 0,0,0,0
S               // Read sector
P               // Print if successful

// Write test data to sector 1000
232 3 0 0 A     // LBA = 1000 (0x03E8)
B               // Fill with pattern (outside menu)
T               // Write sector
```

## Notes

1. **Port Addresses**: Uses same ports as original assembly ($40-$48)
2. **Error Handling**: Returns 0 for failure, non-zero for success
3. **Buffer Management**: 512-byte buffer allocated with `/A` operator
4. **Timeouts**: Implemented with nested counters for long delays
5. **Drive Selection**: Bit 0 of variable `a` selects master(0) or slave(1)
6. **LBA Mode**: Always uses LBA addressing mode (not CHS)

## Limitations

- No interrupt support (polling only)
- Simplified timeout handling compared to assembly version
- Error codes simplified to boolean success/failure
- Buffer access is byte-oriented rather than optimized word access
