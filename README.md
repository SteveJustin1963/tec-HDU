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

////

# 

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

This MINT version maintains the core functionality while adapting to MINT's capabilities and constraints.
 


