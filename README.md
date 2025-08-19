# tec-HDU 

tec1 to ATA HDU 


# Read-in: what design and code do per web documents

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

Looking at your Z80 assembly IDE code and the MINT manual, convert this to run on the TEC-1. This will be a significant conversion since we're going from low-level assembly to MINT's higher-level operations.I've converted your Z80 assembly IDE interface code to MINT for the TEC-1. Here are the key aspects of the conversion:

# User Manual Into - see wiki for code 

## **Complete Coverage:**

### **For Beginners:**
- Step-by-step hardware setup with wiring diagrams
- Software installation procedures  
- Quick start guide with first session examples
- Detailed menu explanations with screenshots
- Common troubleshooting solutions

### **For Advanced Users:**
- Direct function reference for command-line use
- Custom script examples and batch operations
- Performance optimization techniques
- Technical specifications and timing data
- Programming integration examples

### **Key Sections:**

1. **Hardware Setup** - Port mapping, wiring, timing requirements
2. **Menu System** - Complete reference for all 15 menu options  
3. **Memory Operations** - TEC-1 memory layout and transfer procedures
4. **File System** - Simple 2-character filename system usage
5. **Troubleshooting** - Diagnostic procedures and error recovery
6. **Technical Reference** - Function tables, variable usage, performance data

### **Practical Features:**
- **Error message reference** with solutions
- **Quick reference card** for common commands  
- **Sector allocation suggestions** for organizing your disk
- **Programming examples** for automation
- **Capacity calculations** for different drive sizes


# TEC-1 MINT IDE Interface - User Manual

**Version 1.0**  
**For TEC-1 Computer with MINT Interpreter**

---

## Table of Contents

1. [Introduction](#introduction)
2. [System Requirements](#system-requirements)
3. [Hardware Setup](#hardware-setup)
4. [Software Installation](#software-installation)
5. [Getting Started](#getting-started)
6. [Main Menu Reference](#main-menu-reference)
7. [Memory Operations](#memory-operations)
8. [File System Operations](#file-system-operations)
9. [Advanced Usage](#advanced-usage)
10. [Troubleshooting](#troubleshooting)
11. [Technical Reference](#technical-reference)
12. [Appendices](#appendices)

---

## Introduction

The TEC-1 MINT IDE Interface provides a complete storage solution for your TEC-1 computer, allowing you to:

- **Store and retrieve programs** to/from IDE/ATA hard drives or CompactFlash cards
- **Backup and restore** TEC-1 memory contents
- **Create a simple file system** for organizing your programs
- **Transfer data** between TEC-1 memory and mass storage
- **Access modern storage devices** through the standard IDE interface

This system consists of hardware (IDE interface circuit) and software (MINT program) that work together to provide reliable mass storage capabilities.

### Key Features

- ✅ **Raw sector access** for direct disk operations
- ✅ **Memory transfer functions** for backup/restore
- ✅ **Simple file system** with 2-character filenames
- ✅ **Master/Slave drive support** for multiple devices
- ✅ **User-friendly menu system** with guided operations
- ✅ **Error handling and timeout protection**
- ✅ **Progress indicators** for long operations
- ✅ **Hex dump capabilities** for debugging

---

## System Requirements

### Hardware Requirements

**Essential:**
- TEC-1 Computer with Z80 CPU
- MINT Interpreter (SJ MINT) installed
- IDE Interface Hardware (see Hardware Setup)
- IDE/ATA device (hard drive or CompactFlash card)
- Power supply for IDE device (if required)

**Recommended:**
- CompactFlash card (more reliable than old hard drives)
- CF-to-IDE adapter
- 5V power supply for drives requiring it

### Software Requirements

- **MINT Interpreter** version compatible with:
  - 16-bit integer arithmetic
  - Port I/O operations (`/I`, `/O`)
  - Memory allocation (`/A`)
  - All standard MINT operations

### Memory Requirements

- **2KB minimum** free RAM for MINT program
- **512 bytes** allocated for sector buffer
- **Additional space** for file table (if using file system)

---

## Hardware Setup

### IDE Interface Circuit

The IDE interface connects your TEC-1 to standard IDE devices using ports $40-$48:

**Port Mapping:**
```
$40 (64)  - IDE Data Register (16-bit, LSB)
$41 (65)  - IDE Error/Features Register  
$42 (66)  - IDE Sector Count Register
$43 (67)  - IDE LBA 0 (bits 0-7)
$44 (68)  - IDE LBA 1 (bits 8-15)
$45 (69)  - IDE LBA 2 (bits 16-23)
$46 (70)  - IDE Drive/Head Register (LBA 3 + flags)
$47 (71)  - IDE Status/Command Register
$48 (72)  - IDE Data Register (16-bit, MSB latch)
```

### Physical Connections

**IDE 40-Pin Connector Pinout:**
- Pins 1-20: Signal lines (data, address, control)
- Pins 21-40: Ground and power
- Pin 20: Ground (key pin, usually blocked)

**Critical Signals:**
- **Data Bus D0-D15**: Connect to TEC-1 data bus via latches
- **Address A0-A2**: Register select lines from TEC-1
- **CS0/CS1**: Chip select (decoded from TEC-1 address)
- **IOR/IOW**: Read/Write strobes from TEC-1
- **Reset**: Connected to TEC-1 reset line

### Timing Considerations

The interface must meet IDE timing requirements:
- **Setup time**: 70ns (mode 0) to 10ns (mode 6)
- **Hold time**: 20ns minimum
- **Cycle time**: 600ns (mode 0) to 25ns (mode 6)

TEC-1's Z80 running at 1-4MHz provides adequate timing margins for PIO modes 0-2.

### Device Configuration

**Jumper Settings:**
- **Single drive**: Set as Master (usually default)
- **Two drives**: Set one as Master, one as Slave
- **CompactFlash**: Usually defaults to Master, no jumpers needed

**Power Requirements:**
- Most modern CF cards: 3.3V/5V (powered from TEC-1)
- Older hard drives: 5V + 12V (external supply required)

---

## Software Installation

### Loading the MINT Code

1. **Connect to TEC-1** via serial terminal at 4800 baud
2. **Start MINT** on your TEC-1
3. **Upload the code** in sections to avoid buffer overflow

**Upload Procedure:**
```
> // Copy and paste core functions first
> // Test basic functionality
> // Upload menu system
> // Test complete system
```

### Initial Setup Sequence

```mint
// 1. Load all functions (copy/paste from code)
// 2. Initialize the system
I

// 3. Test basic connectivity  
G

// 4. If successful, start menu system
O
```

### Verification Steps

**Test 1: Drive Detection**
```mint
I         // Initialize
G         // Get drive ID
```
*Expected: Drive model and serial number displayed*

**Test 2: Sector Access**
```mint
0 V       // Select master
0 0 0 0 A // Set LBA to 0
S         // Read sector
```
*Expected: Success message, no timeout*

**Test 3: Memory Transfer**
```mint
#8000 256 F  // Load 256 bytes from $8000
#9000 256 Z  // Save to $9000
```
*Expected: Data copied between memory locations*

---

## Getting Started

### Quick Start Guide

**Step 1: Hardware Check**
- Verify IDE interface is properly connected
- Ensure drive power (if required)
- Check all cables and jumpers

**Step 2: Software Setup**
- Load MINT IDE code into TEC-1
- Run initialization: `I`
- Test drive detection: `G`

**Step 3: First Operations**
- Read a sector: Menu option 4
- Dump sector contents: Menu option 6
- Try memory transfer: Menu option A

### First Session Example

```
> I                    // Initialize system
IDE System Initialized

> G                    // Get drive info
Drive ID Read Successfully
Model: TRANSCEND       
Serial: 12345678       

> O                    // Start menu
=====================================
        TEC-1 IDE Interface          
=====================================
1. Initialize IDE System
2. Select Drive (Master/Slave)
...
Enter choice: 4        // Read sector

Enter LBA address (4 bytes, LSB first):
LBA0 (0-255): 0
LBA1 (0-255): 0
LBA2 (0-255): 0
LBA3 (0-255): 0

Reading sector...
Sector read successfully
First 64 bytes:
EB 58 90 4D 53 44 4F 53 35 2E 30 00 02 08 01 00
...
```

### Common First Tasks

**1. Backup a TEC-1 Program**
```
Menu A: Load Memory to Buffer
Start address: 8000    // Your program location
Byte count: 200        // Program size

Menu 5: Write Sector   
LBA: 0 0 10 0         // Sector 1000
Confirm write: 1       // Yes
```

**2. Create Test File**
```
Menu C: Write File from Memory
Filename: TE           // "TEST" 
Start address: 8000
Length: 512

Writing 1 sectors starting at LBA...
File written successfully
```

**3. Restore Program**
```
Menu 4: Read Sector
LBA: 0 0 10 0         // Sector 1000

Menu B: Save Buffer to Memory  
Start address: 9000    // Different location
Byte count: 200
```

---

## Main Menu Reference

### Menu Layout

```
=====================================
        TEC-1 IDE Interface          
=====================================
1. Initialize IDE System
2. Select Drive (Master/Slave)
3. Get Drive Information
4. Read Sector
5. Write Sector
6. Dump Sector (Hex)
7. Fill Buffer with Pattern
8. Show Current LBA
9. Set LBA Address
A. Load Memory to Buffer
B. Save Buffer to Memory
C. Write File from Memory
D. Read File to Memory
E. List Files
0. Exit
=====================================
```

### Option 1: Initialize IDE System

**Purpose:** Sets up the IDE interface and allocates sector buffer

**Process:**
- Clears IDE status variables
- Allocates 512-byte sector buffer
- Resets LBA registers to zero
- Prepares system for operations

**Usage:**
- Run this first after loading the code
- Re-run if system becomes unstable
- No user input required

**Example:**
```
Enter choice: 1
IDE System Initialized
```

### Option 2: Select Drive (Master/Slave)

**Purpose:** Choose which drive to access (if multiple drives connected)

**Process:**
- Displays drive selection menu
- Updates drive select bit in IDE status
- Affects all subsequent operations

**Usage:**
- 0 = Master drive (primary)
- 1 = Slave drive (secondary)
- Most single-drive systems use Master

**Example:**
```
Enter choice: 2
Select Drive:
0 = Master Drive
1 = Slave Drive
Enter choice: 0
Drive 0 selected
```

### Option 3: Get Drive Information

**Purpose:** Reads and displays drive identification data

**Information Displayed:**
- Drive model name (40 characters)
- Serial number (20 characters)
- Internal drive parameters

**Process:**
- Sends IDE IDENTIFY command ($EC)
- Reads 512 bytes of drive information
- Extracts and displays key fields

**Example:**
```
Enter choice: 3
Getting drive information...
Drive ID Read Successfully
Model: TRANSCEND CF CARD                 
Serial: 20190405001234
```

**Troubleshooting:**
- If fails: Check connections, power, drive jumpers
- Timeout: Drive may not be ready or compatible
- Garbled text: Possible timing or wiring issues

### Option 4: Read Sector

**Purpose:** Read any sector from the drive to the buffer

**Input Required:**
- LBA address (4 bytes: LBA0, LBA1, LBA2, LBA3)
- Values 0-255 for each byte

**Process:**
- Sets up LBA registers
- Sends READ SECTOR command ($20)
- Transfers 512 bytes to sector buffer
- Displays first 64 bytes as preview

**Example:**
```
Enter choice: 4
Enter LBA address (4 bytes, LSB first):
LBA0 (0-255): 0
LBA1 (0-255): 0  
LBA2 (0-255): 0
LBA3 (0-255): 0

Reading sector...
Sector read successfully
First 64 bytes:
EB 58 90 4D 53 44 4F 53 35 2E 30 00 02 08 01 00
02 00 02 00 00 F8 00 00 3F 00 FF 00 00 00 00 00
00 00 00 00 00 00 29 A1 B2 C3 D4 4E 4F 20 4E 41
4D 45 20 20 20 20 46 41 54 31 36 20 20 20 0E 1F
```

### Option 5: Write Sector

**Purpose:** Write the current buffer contents to any sector

**Input Required:**
- LBA address (4 bytes)
- Confirmation (safety feature)

**Warning:** This overwrites data permanently!

**Process:**
- Sets up LBA registers  
- Requests user confirmation
- Sends WRITE SECTOR command ($30)
- Transfers buffer contents to drive
- Verifies completion

**Example:**
```
Enter choice: 5
WARNING: This will overwrite data!
Enter LBA address (4 bytes, LSB first):
LBA0 (0-255): 0
LBA1 (0-255): 0
LBA2 (0-255): 10
LBA3 (0-255): 0

Are you sure? (1=Yes, 0=No): 1
Writing sector...
Sector written successfully
```

### Option 6: Dump Sector (Hex)

**Purpose:** Display current buffer contents in hexadecimal format

**Features:**
- Shows all 512 bytes
- 16 bytes per line
- Hexadecimal format
- Useful for debugging and verification

**Example:**
```
Enter choice: 6
Sector data:
00: EB 58 90 4D 53 44 4F 53 35 2E 30 00 02 08 01 00
10: 02 00 02 00 00 F8 00 00 3F 00 FF 00 00 00 00 00
20: 00 00 00 00 00 00 29 A1 B2 C3 D4 4E 4F 20 4E 41
...
```

### Option 7: Fill Buffer with Pattern

**Purpose:** Fill the sector buffer with test patterns

**Available Patterns:**
1. **Sequential** (0,1,2,3...255,0,1...)
2. **Alternating** (0xAA,0x55,0xAA,0x55...)  
3. **All zeros** (0x00 repeated)
4. **All ones** (0xFF repeated)

**Usage:**
- Useful for testing drive functionality
- Creates known data patterns
- Helps verify read/write operations

**Example:**
```
Enter choice: 7
Select pattern:
1. Sequential (0,1,2,3...)
2. Alternating (0xAA,0x55...)
3. All zeros
4. All ones (0xFF)
Enter choice: 1
Buffer filled with sequential pattern
```

### Option 8: Show Current LBA

**Purpose:** Display the current LBA address setting

**Information Shown:**
- All four LBA bytes (LBA0-LBA3)
- Current sector number calculation
- Useful for tracking location

**Example:**
```
Enter choice: 8
Current LBA: 0 0 10 0
Sector: 2560 (0x0A00)
```

### Option 9: Set LBA Address

**Purpose:** Manually set the target sector address

**Input Required:**
- Four LBA bytes (0-255 each)
- LBA0 = bits 0-7 (LSB)
- LBA1 = bits 8-15
- LBA2 = bits 16-23  
- LBA3 = bits 24-27 (MSB)

**Calculation:**
Sector Number = LBA3×16777216 + LBA2×65536 + LBA1×256 + LBA0

**Example:**
```
Enter choice: 9
Current LBA: 0 0 0 0
Enter new LBA (4 bytes):
LBA0 (LSB): 0
LBA1:      0
LBA2:      10  
LBA3 (MSB): 0
LBA set to: 0 0 10 0
```

### Option A: Load Memory to Buffer

**Purpose:** Copy data from TEC-1 memory to the sector buffer

**Input Required:**
- Start address (in hexadecimal)
- Byte count (1-512)

**Process:**
- Reads bytes from TEC-1 memory
- Stores in sector buffer
- Prepares for writing to disk

**Example:**
```
Enter choice: A
Memory to Buffer Transfer
Start address (hex): 8000
Byte count (max 512): 256
Loading 256 bytes from address 8000
Memory loaded to buffer
```

### Option B: Save Buffer to Memory

**Purpose:** Copy sector buffer contents to TEC-1 memory

**Input Required:**
- Destination address (in hex)
- Byte count (1-512)

**Process:**
- Reads bytes from sector buffer
- Writes to TEC-1 memory
- Useful for restoring programs

**Example:**
```
Enter choice: B
Buffer to Memory Transfer  
Start address (hex): 9000
Byte count (max 512): 256
Saving 256 bytes to address 9000
Buffer saved to memory
```

### Option C: Write File from Memory

**Purpose:** Save TEC-1 memory contents as a multi-sector "file"

**Input Required:**
- Filename (2 characters)
- Start address (hex)
- Length (bytes)

**Process:**
- Calculates sectors needed
- Transfers memory in 512-byte chunks
- Writes consecutive sectors
- Updates file table (future feature)

**Example:**
```
Enter choice: C
Enter filename (2 chars): PR
Start address (hex): 8000
Length (bytes): 1024
Writing 2 sectors starting at LBA 0 0 0 0
..
File written successfully
```

### Option D: Read File to Memory

**Purpose:** Load a stored "file" directly into TEC-1 memory

**Input Required:**
- Filename (2 characters)
- Destination address (hex)
- Start LBA and sector count (current implementation)

**Process:**
- Reads consecutive sectors
- Transfers to TEC-1 memory
- Reconstructs original data

**Example:**
```
Enter choice: D
Enter filename (2 chars): PR
Destination address (hex): 9000
Start LBA: 0 0 0 0
Length (sectors): 2
Reading 2 sectors to address 9000
..
File read successfully
```

### Option E: List Files

**Purpose:** Display stored files (basic implementation)

**Current Status:** Placeholder for future file table system

**Future Features:**
- File directory listing
- File sizes and dates
- Free space calculation

### Option 0: Exit

**Purpose:** Exit the menu system and return to MINT prompt

**Process:**
- Displays goodbye message
- Returns to MINT interpreter
- System remains initialized

---

## Memory Operations

### Understanding TEC-1 Memory Layout

**Typical TEC-1 Memory Map:**
```
$0000-$07FF   ROM (MINT Interpreter)
$0800-$0FFF   System Variables  
$1000-$17FF   User RAM
$1800-$1FFF   Stack Area
$2000-$7FFF   Extended RAM (if available)
$8000-$FFFF   I/O Space / Extended Memory
```

**Safe Areas for Programs:**
- `$1000-$17FF`: Basic user programs
- `$2000-$7FFF`: Large programs (if RAM available)
- `$8000-$9FFF`: Monitor programs

### Memory Transfer Examples

**Example 1: Backup Small Program**
```mint
// Program at $8000, 200 bytes
Menu A: Load Memory to Buffer
Start address: 8000
Byte count: 200

Menu 5: Write Sector
LBA: 100 0 0 0    // Sector 100
Confirm: 1
```

**Example 2: Backup Large Program** 
```mint
// Program at $2000, 2048 bytes (4 sectors)
Menu C: Write File from Memory  
Filename: BG        // "BIG"
Start address: 2000
Length: 2048
// Automatically writes 4 sectors
```

**Example 3: Memory-to-Memory Copy**
```mint
// Copy data using disk as intermediate
Menu A: Load from $8000, 512 bytes
Menu B: Save to $9000, 512 bytes
// 512 bytes copied from $8000 to $9000
```

### Memory Safety Considerations

**Avoid These Areas:**
- `$0000-$07FF`: ROM (read-only anyway)
- `$0800-$0FFF`: MINT variables
- `$1F00-$1FFF`: Stack (active during operations)

**Safe Practices:**
- Always specify exact byte counts
- Don't exceed 512 bytes per transfer
- Test with small amounts first
- Keep backups of important programs

---

## File System Operations

### Simple File System Overview

The TEC-1 MINT IDE interface implements a basic file system:

**Features:**
- 2-character filenames (e.g., "PR", "DT", "GM")
- Sequential sector allocation
- Automatic size calculation
- No subdirectories
- No file fragmentation

**Limitations:**
- No file table persistence (planned feature)
- No file deletion (manual sector overwrite)
- No file dates or attributes
- Maximum filename length: 2 characters

### File Naming Conventions

**Suggested Naming:**
- **Programs**: PR, PG, AP (Application)
- **Data**: DT, DA, DB (Database)
- **Games**: GM, G1, G2
- **Tools**: TL, UT (Utility)
- **Tests**: TS, T1, T2

**Examples:**
```
Filename: Purpose
PR       Main program
DT       Data file  
GM       Game
T1       Test program 1
SV       Saved state
BK       Backup
```

### File Operation Examples

**Example 1: Save Game High Scores**
```mint
// High scores at $1500, 100 bytes
Menu C: Write File from Memory
Filename: HS        // "High Scores"
Start address: 1500
Length: 100
```

**Example 2: Create Program Library**
```mint
// Save multiple programs
Filename: P1, Start: $8000, Length: 512
Filename: P2, Start: $8200, Length: 300  
Filename: P3, Start: $8400, Length: 256
```

**Example 3: Backup System**
```mint
// Daily backup routine
Filename: MO        // Monday
Start: $1000
Length: 2048        // Full user area

// Restore if needed:
Filename: MO
Destination: $2000  // Test area first
```

### File Management Strategies

**Organization:**
- Use consistent naming scheme
- Document file contents separately
- Keep track of LBA locations manually
- Create file catalog on paper/computer

**Backup Strategy:**
- Multiple copies of important files
- Regular backup schedule
- Test restore procedures
- Keep development and production versions

---

## Advanced Usage

### Direct Function Access

All menu functions can be called directly from MINT command line:

**Core Functions:**
```mint
I         // Initialize system
G         // Get drive ID
S         // Read sector  
T         // Write sector
L         // Setup LBA
W         // Wait for ready
E         // Test error
D         // Wait for data
R         // Read buffer
Y         // Write buffer
```

**Memory Functions:**
```mint
#8000 256 F    // Load 256 bytes from $8000
#9000 256 Z    // Save 256 bytes to $9000
```

**LBA Functions:**
```mint
0 0 0 1 A      // Set LBA to sector 256 (1,0,0,0)
X              // Show current LBA
```

### Batch Operations

**Example: Multi-Sector Read**
```mint
// Read 4 consecutive sectors
0 0 0 0 A  S   // Sector 0
1 0 0 0 A  S   // Sector 1  
2 0 0 0 A  S   // Sector 2
3 0 0 0 A  S   // Sector 3
```

**Example: Memory Dump**
```mint
// Dump memory in 512-byte chunks
#1000 512 F  P    // Dump $1000-$11FF
#1200 512 F  P    // Dump $1200-$13FF  
#1400 512 F  P    // Dump $1400-$15FF
```

### Custom Scripts

Create MINT functions for common operations:

**Backup Function:**
```mint
:4                  // Custom backup function
8000 512 F         // Load from $8000
100 0 0 0 A        // Set to sector 100
T                  // Write
;

// Usage: 4  (backs up $8000 to sector 100)
```

**Restore Function:**
```mint
:6                  // Custom restore function  
100 0 0 0 A        // Set to sector 100
S                  // Read
8000 512 Z         // Save to $8000
;

// Usage: 6  (restores sector 100 to $8000)
```

### Performance Optimization

**Fast Sector Access:**
- Keep frequently used LBA values in variables
- Use direct function calls instead of menu
- Batch multiple operations together

**Memory Management:**
- Allocate buffer once at startup
- Reuse buffer for multiple operations
- Clear buffer between different data types

**Error Recovery:**
- Always check return values
- Implement retry logic for failures
- Have fallback procedures

### Integration with Other TEC-1 Software

**Monitor Integration:**
- Save/restore monitor programs
- Backup system configuration
- Store custom monitor extensions

**BASIC Integration:**
- Save BASIC programs to disk
- Load program libraries
- Create data file systems

**Assembly Integration:**
- Store assembled code
- Backup development versions
- Create code libraries

---

## Troubleshooting

### Common Problems and Solutions

#### Problem: "Drive ID Failed"

**Possible Causes:**
- Hardware not connected
- Incorrect wiring
- Power supply issues
- Drive not configured properly

**Solutions:**
1. Check all connections
2. Verify power to drive
3. Check jumper settings (Master/Slave)
4. Try different drive
5. Verify port addresses in code

**Testing:**
```mint
// Test basic port access
71 /I .    // Should return drive status
// If returns 255 or 0, check hardware
```

#### Problem: "Sector Read Failed" 

**Possible Causes:**
- Bad sector on drive
- Timeout issues
- LBA address out of range
- Drive not ready

**Solutions:**
1. Try different sector: `0 0 0 0 A S`
2. Check drive capacity
3. Verify LBA calculation
4. Test with known good sector

**Debugging:**
```mint
// Check status after failed read
71 /I ,    // Display status in hex
65 /I ,    // Display error register
```

#### Problem: "Write Failed"

**Possible Causes:**
- Write-protected drive/media
- Bad sector
- Drive full
- Timeout

**Solutions:**
1. Check write-protect jumper
2. Try different sector
3. Verify drive has free space
4. Test with pattern write first

#### Problem: "Timeout Errors"

**Possible Causes:**
- Drive too slow to respond
- Clock speed too fast
- Marginal hardware

**Solutions:**
1. Add delays to timeout loops
2. Reduce TEC-1 clock speed
3. Check power supply stability
4. Try different drive

**Code Modification:**
```mint
// Increase timeout in function W
// Change: 50 ( ) to 100 ( )
// Or add extra delays: 10 ( )
```

#### Problem: "Memory Transfer Errors"

**Possible Causes:**
- Invalid memory addresses
- Buffer overflow
- Stack corruption

**Solutions:**
1. Verify address ranges
2. Limit transfer sizes
3. Check available RAM
4. Restart MINT if corrupted

#### Problem: "Garbled Data"

**Possible Causes:**
- Timing issues
- Endian problems
- Buffer corruption
- Hardware noise

**Solutions:**
1. Add delays to read/write
2. Check data immediately after transfer
3. Use test patterns to verify
4. Check ground connections

### Diagnostic Procedures

**Hardware Test:**
```mint
// 1. Basic port test
64 /I .    // Read data port
71 /I .    // Read status port

// 2. Drive detection  
I G        // Initialize and get ID

// 3. Simple read test
0 0 0 0 A  // Set LBA 0
S          // Read sector
P          // Display contents
```

**Data Integrity Test:**
```mint
// 1. Fill buffer with pattern
7          // Menu: Fill buffer
1          // Sequential pattern

// 2. Write to test sector
5          // Menu: Write sector
255 255 255 0  // High LBA (test area)

// 3. Clear buffer and read back
7 3        // Fill with zeros
4          // Read same sector
6          // Check if pattern restored
```

**Memory Test:**
```mint
// 1. Load known data
#8000 256 F    // From program area

// 2. Save to different location  
#9000 256 Z    // To free area

// 3. Compare manually or with monitor
```

### Error Code Reference

**IDE Status Register (Port $47):**
- Bit 7: BSY (Busy)
- Bit 6: RDY (Ready)  
- Bit 5: DF (Drive Fault)
- Bit 4: DSC (Seek Complete)
- Bit 3: DRQ (Data Request)
- Bit 2: CORR (Corrected Data)
- Bit 1: IDX (Index)
- Bit 0: ERR (Error)

**IDE Error Register (Port $41) when ERR=1:**
- Bit 7: BBK (Bad Block)
- Bit 6: UNC (Uncorrectable)
- Bit 5: MC (Media Changed)
- Bit 4: IDNF (ID Not Found)
- Bit 3: MCR (Media Change Request)
- Bit 2: ABRT (Aborted Command)
- Bit 1: TK0NF (Track 0 Not Found)
- Bit 0: AMNF (Address Mark Not Found)

### Recovery Procedures

**System Recovery:**
1. Power cycle TEC-1 and drive
2. Reload MINT IDE code
3. Re-initialize: `I`
4. Test basic functions

**Data Recovery:**
```mint
// If file system corrupted:
// 1. Try reading known sector addresses
0 0 0 0 A S    // Try sector 0
1 0 0 0 A S    // Try sector 1

// 2. Search for your data
// Use menu option 6 to examine sectors
// Look for recognizable patterns

// 3. Manual sector recovery
// Read each sector and save to memory
// Reconstruct files manually
```

---

## Technical Reference

### MINT Function Reference

**Core IDE Functions:**
| Function | Purpose | Parameters | Returns |
|----------|---------|------------|---------|
| `I` | Initialize system | None | None |
| `G` | Get drive ID | None | Success/Fail |
| `S` | Read sector | LBA set | Success/Fail |
| `T` | Write sector | LBA set, buffer loaded | Success/Fail |
| `L` | Setup LBA | None (uses c,d,e,f vars) | None |
| `W` | Wait for ready | None | Success/Fail |
| `E` | Test for errors | None | Success/Fail |
| `D` | Wait for data | None | Success/Fail |
| `R` | Read buffer | None | None |
| `Y` | Write buffer | None | None |

**Memory Functions:**
| Function | Purpose | Parameters | Returns |
|----------|---------|------------|---------|
| `F` | Load memory to buffer | address, count | None |
| `Z` | Save buffer to memory | address, count | None |
| `4` | Input hex address | None | 16-bit value |

**Utility Functions:**
| Function | Purpose | Parameters | Returns |
|----------|---------|------------|---------|
| `A` | Set LBA address | lba3,lba2,lba1,lba0 | None |
| `V` | Set drive select | 0=master, 1=slave | None |
| `P` | Print buffer contents | None | None |
| `X` | Show current LBA | None | None |
| `M` | Master/slave select helper | value | modified value |

### Variable Usage

**Global Variables:**
- `a` - IDE status/drive select
- `b` - Sector buffer pointer
- `c` - LBA byte 0 (LSB)
- `d` - LBA byte 1
- `e` - LBA byte 2  
- `f` - LBA byte 3 (MSB)
- `g`-`z` - Temporary storage

**System Variables:**
- `/i` - Loop counter (inner)
- `/j` - Loop counter (outer)
- `/c` - Carry flag
- `/r` - Remainder/overflow
- `/s` - Stack pointer

### Memory Map

**MINT IDE System:**
```
Heap:     Sector buffer (512 bytes)
Variables: a-z (52 bytes)  
Code:     Function definitions (~2KB)
Stack:    MINT interpreter stack
```

**Recommended Memory Layout:**
```
$1000-$13FF   User programs (1KB)
$1400-$15FF   Data storage (512B)
$1600-$17FF   IDE buffer area (512B)
$1800-$1FFF   Stack space (2KB)
```

### Performance Characteristics

**Timing (approximate, TEC-1 @ 2MHz):**
- Sector read: 1-2 seconds
- Sector write: 1-2 seconds  
- Memory transfer (512B): 0.5 seconds
- Drive ID: 1 second
- Buffer operations: 0.1 seconds

**Throughput:**
- Sequential read: ~256 bytes/second
- Sequential write: ~256 bytes/second
- Random access: ~128 bytes/second (including seek)

### Capacity Calculations

**LBA Address Space:**
- 28-bit LBA = 268,435,456 sectors
- 512 bytes/sector = 137 GB maximum
- Practical limit: 2GB (older drives)

**Common Drive Sizes:**
| Capacity | Sectors | LBA Range |
|----------|---------|-----------|
| 16 MB | 32,768 | 0x0000-0x7FFF |
| 32 MB | 65,536 | 0x0000-0xFFFF |
| 64 MB | 131,072 | 0x0000-0x1FFFF |
| 128 MB | 262,144 | 0x0000-0x3FFFF |
| 256 MB | 524,288 | 0x0000-0x7FFFF |

### Hardware Interface Details

**Port Usage:**
```
$40: Data LSB (read/write)
$41: Error/Features 
$42: Sector count (always 1)
$43: LBA bits 0-7
$44: LBA bits 8-15  
$45: LBA bits 16-23
$46: Drive/LBA bits 24-27
$47: Status/Command
$48: Data MSB latch
```

**Command Codes:**
- `$20` - Read Sectors
- `$30` - Write Sectors
- `$EC` - Identify Drive
- `$E7` - Flush Cache

---

## Appendices

### Appendix A: Error Messages

| Message | Cause | Solution |
|---------|-------|----------|
| "Drive ID Failed" | No drive response | Check hardware |
| "Sector read failed" | Read timeout/error | Try different sector |
| "Sector write failed" | Write timeout/error | Check write protection |
| "Invalid choice" | Bad menu input | Enter valid option |
| "Buffer filled with..." | Success message | None required |
| "Memory loaded to buffer" | Success message | None required |
| "File written successfully" | Success message | None required |

### Appendix B: Quick Reference Card

**Essential Commands:**
```
I          Initialize system
O          Start menu
G          Get drive info  
0 0 0 0 A  Set LBA to 0
S          Read sector
P          Print buffer
```

**Memory Operations:**
```
#8000 256 F    Load from $8000
#9000 256 Z    Save to $9000
```

**File Operations:**
```
Menu C: Save file from memory
Menu D: Load file to memory
```

### Appendix C: Sector Allocation Map

**Suggested Layout:**
```
Sectors 0-99:       System/boot area
Sectors 100-999:    User programs  
Sectors 1000-1999:  Data files
Sectors 2000-2999:  Backups
Sectors 3000+:      Free space
```

### Appendix D: Programming Examples

**Example 1: Automatic Backup**
```mint
:7                    // Daily backup function
`Daily backup starting...` /N
#1000 2048 F         // Load user area
100 /i + 0 0 0 A     // Sector 100+day
T                    // Write
`Backup complete` /N
;
```

**Example 2: File Search**
```mint
:8                    // Search for pattern
`Searching...` /N
100 (                // Search 100 sectors
  /i 0 0 0 A         // Set LBA
  S                  // Read sector
  // Check for pattern in buffer
  /i .               // Show progress
)
;
```

**Example 3: Disk Utility**
```mint
:9                    // Disk information
G                     // Get drive ID
`Checking sectors...` /N
0 k !                 // Error count
100 (                 // Check 100 sectors
  /i 0 0 0 A
  S 0 = ( k 1 + k ! ) // Count errors
)
`Errors found: ` k . /N
;
```

### Appendix E: Troubleshooting Flowchart

```
Start
  |
  v
Drive responds to ID command?
  |NO -> Check hardware connections
  |      Check power supply
  |      Verify jumper settings
  |      
  |YES
  v
Can read sector 0?
  |NO -> Try different sectors
  |      Check LBA calculation
  |      Reduce timeout values
  |
  |YES  
  v
Can write to test sector?
  |NO -> Check write protection
  |      Try different sectors
  |      Verify drive capacity
  |
  |YES
  v
Memory transfers work?
  |NO -> Check address ranges
  |      Verify buffer allocation
  |      Test with small amounts
  |
  |YES
  v
System working normally
```

---

## Conclusion

The TEC-1 MINT IDE Interface provides a robust storage solution for your TEC-1 computer. With proper setup and understanding of the commands, you can:

- **Backup and restore** programs reliably
- **Create a personal library** of TEC-1 software
- **Transfer data** between memory and storage
- **Experiment with mass storage** programming

For additional support or to report issues, consult the TEC-1 community forums or reference the original MINT documentation.

**Happy computing with your TEC-1 IDE system!**

---



