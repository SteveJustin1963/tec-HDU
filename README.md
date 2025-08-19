# tec-HDU

used chatgpt and claude

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

What I will port to MINT (preview plan, single-letter functions, small blocks):

* \:D set drive bit in ide\_status.
* \:S load LBA0..3 and prime reg2..6.
* \:P poll status (BSY/DRDY/DRQ) with timeouts.
* \:R read one sector into buffer.
* \:W write one sector from buffer.
* \:G run IDENTIFY into buffer.
  All operations will mirror the exact register order and byte order above. &#x20;

If that matches your intent, I’ll start mapping each routine to minimal MINT blocks next.



