# skiroc2-slow-control

## Introduction

This page documents the software used to access the SKIROC2 slow control (SC) register.
In the context of PetaLinux, the 'server' is the Linux OS itself; and the functions that read/write registers are built as different executable programs.
The software currently consists of these programs:

- `sk2_sc_peek`
  - Read the value of a given register.
- `sk2_sc_poke`
  - Write a value into a given register.
- `sk2_sc_reset`
  - Reset all the registers to their default values.
- `sk2_sc_scan`
  - Read the values of all the registers in sequence.
- `sk2_sc_bitbang`
  - Output the values of all the registers serially by doing software bit-banging.
- `sk2_sc_bitbang_ipbus`
  - Output the values of all the registers serially by doing software bit-banging, using a simplified IPbus protocol.

The software design is explained in section [Software Design](#software-design). The program usage is described in section [Program Usage](#program-usage).

### Software Design

The software has been uploaded to [test_beam_DAQ_2015](https://github.com/jbueghly/test_beam_DAQ_2015/) GitHub repository.
The software can be found in the directory [projects/testbeam_5b/software/sk2_sc_peekpoke](https://github.com/jbueghly/test_beam_DAQ_2015/tree/master/projects/testbeam_5b/software/sk2_sc_peekpoke).

The names and other properties of the registers in SKIROC2 SC are hard-coded in the python script `define_SKIROC2_SC.py` like the following:

```python
skiroc2_slow_control_register = [
# Tuple format: (register name, number of bits, subaddress, default value, number of channels)
("EN_PA"                               ,   1,   0,          0b1,  1),
("PP_PA"                               ,   1,   1,          0b0,  1),
('GC_PA_Comp'                          ,   3,   2,        0b111,  1),
...
('EC_Dout2'                            ,   1, 614,          0b1,  1),
('EC_Dout1'                            ,   1, 615,          0b1,  1),
]
```

The python script `generate_SKIROC2_SC.py`, when called, generates a C header file `SKIROC2_SC_hw.h`, with a lookup table like the following. 
Note that the lookup table entries are sorted by the register name to allow binary search and reduce lookup time.

```python
static const item_t lookup_table[LOOKUP_TABLE_SIZE+1] = {  // plus one NULL entry
    {"DA_CH00", 4, 223, 1},
    {"DA_CH01", 4, 227, 1},
    {"DA_CH02", 4, 231, 1},
...
    {"TM_CH63", 1, 542, 0},
    {NULL, 0, 0, 0}
};
```

All the important functions are declared in `SKIROC2.h` and implemented in `SKIROC2.c`. For instance:

```c
// Look up using binary search
const item_t * skiroc2_slow_control_lookup_item(const char *key);

// Get and set functions
void skiroc2_slow_control_set(const item_t * item, unsigned int value);

unsigned int skiroc2_slow_control_get(const item_t * item);

// Peek and poke functions
void skiroc2_slow_control_poke(const char *key, unsigned int value);

unsigned int skiroc2_slow_control_peek(const char *key);

void skiroc2_slow_control_scan();

// Init functions
void skiroc2_slow_control_set_default();

void skiroc2_slow_control_init();  // from control_register_file
```

A simplified subset of IPbus protocol is declared in `IPbus_simplified.h` and implemented in `IPbus_simplified.c`. The protocol specifies a packet header format like the following:

```c
// Transport Layer protocol - loosely based on the IPBus protocol
// with the "Transaction ID" field replaed with an "Address" field
//
// | 31 ... 28 | 27 ... 16 | 15 ...  8 |  7 ...  4 |  3 ...  0 |
// |-----------|-----------|-----------|-----------|-----------|
// | Protocol  | Starting  | Number of | Type ID   | Info code |
// | version   | address   | words     |           |           |
// | (4 bits)  | (12 bits) | (8 bits)  | (4 bits)  | (4 bits)  |
//
// - Protocol version: reserved, currently 0
// - Starting address: the intended address for this transaction
// - Number of words : length of the transaction (0-255, 8 bits)
// - Type ID         : 0x0 - read
//                     0x1 - write
// - Info code       : 0x0 - response
//                     0xF - request
```

> :warning: <span style="color: #F00">The IPbus implementation has been updated in [projects/testbeam_6a/software/sk2_sc_transport](https://github.com/jbueghly/test_beam_DAQ_2015/tree/master/projects/testbeam_6a/software/sk2_sc_transport). See page [Transport Layer](transport-layer.md) for its documentation. </span>

The software currently only supports one main transaction (write):

```c
/// Functions
int create_ipbus_write_txn(unsigned int start_addr, int nwords, unsigned int* data_words, unsigned int* buffer);

unsigned int create_ipbus_write_txn_header(unsigned int start_addr, int nwords);

unsigned int create_ipbus_write_txn_data(unsigned int data_word);
```

The executable programs are defined in `sk2_sc_peek.c`, `sk2_sc_poke.c`, `sk2_sc_reset.c`, `sk2_sc_scan.c`, `sk2_sc_bitbang`, `sk2_sc_bitbang_ipbus`. 
See the section [Program Usage](#program-usage) for usage detail. 
They can be built in the PetaLinux environment by using `petalinux-build` with the default makefile `Makefile`. They can also be built on any gcc-compatible Linux OS by using `make` with the makefile `Makefile-gcc`.

`sk2_sc_reset` must be called at least once to initialize all the registers to their default values. The program creates a plain-text file `sk2_sc_cur_state` that records the current 'state' of the SKIROC2 SC register (all those 616 bits). Without the file `sk2_sc_cur_state`, other programs will fail.


### Program Usage

- `sk2_sc_peek`

```
# Usage: sk2_sc_peek REGISTER

$ sk2_sc_peek EN_PA
EN_PA 0b1
$ sk2_sc_peek PP_PA
PP_PA 0b0
```

- `sk2_sc_poke`

```
# Usage: sk2_sc_poke REGISTER VALUE

$ sk2_sc_peek EN_PA
EN_PA 0b1
$ sk2_sc_poke EN_PA 0
EN_PA 0b0
$ sk2_sc_peek EN_PA
EN_PA 0b1
```

- `sk2_sc_reset`

```
# Usage: no arguments

$ sk2_sc_reset
'control_register' has 616 bits encoded in 20 words.
0x00000000 0x00000000 0x00000000 0x00000000 
0x00000000 0x00000000 0x00000000 0x00000000 
0x00000000 0x00000000 0x00000000 0x00000000 
0x00000000 0x00000000 0x00000000 0x00000000 
0x00000000 0x00000000 0x00000000 0x00000000 
'control_register' after initialization with default values.
0x000000FF 0xFFC432FF 0xFFEAC2E0 0x80000000 
0x00000000 0x08888888 0x88888888 0x88888888 
0x88888888 0x88888888 0x88888888 0x88888888 
0x88888888 0xAA922A00 0x00000000 0x00000000 
0x00000000 0x00000000 0x00000000 0x000001FD 
Created file 'sk2_sc_cur_state'.
```

- `sk2_sc_scan`

```
# Usage: no arguments

$ sk2_sc_scan
EN_PA 0b1
PP_PA 0b0
...
EC_DOUT2 0b1
EC_DOUT1 0b1
```

- `sk2_sc_bitbang`

```
# Usage: no arguments

$ sk2_sc_bitbang
Write byte: 0xFF (0b11111111).
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
...
Write byte: 0xFD (0b11111101).
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
-- write to destination: 0x02 (0b00000010)
-- write to destination: 0x06 (0b00000110)
-- write to destination: 0x0A (0b00001010)
-- write to destination: 0x0E (0b00001110)
```

- `sk2_sc_bitbang_ipbus`

```
# Usage: no arguments

$ sk2_sc_bitbang_ipbus
IPbus Packet Header: 0x0000FF1F
  protocol: v0 start_addr: 0x0 nwords: 255 type: 0x1 info: 0xF
IPbus Packet Data   0: 0x0000000A
IPbus Packet Data   1: 0x0000000E
IPbus Packet Data   2: 0x0000000A
IPbus Packet Data   3: 0x0000000E
IPbus Packet Data   4: 0x0000000A
IPbus Packet Data   5: 0x0000000E
IPbus Packet Data   6: 0x0000000A
IPbus Packet Data   7: 0x0000000E
IPbus Packet Data   8: 0x0000000A
IPbus Packet Data   9: 0x0000000E
IPbus Packet Data  10: 0x0000000A
IPbus Packet Data  11: 0x0000000E
IPbus Packet Data  12: 0x0000000A
IPbus Packet Data  13: 0x0000000E
IPbus Packet Data  14: 0x0000000A
IPbus Packet Data  15: 0x0000000E
...
IPbus Packet Data 194: 0x0000000A
IPbus Packet Data 195: 0x0000000E
IPbus Packet Data 196: 0x0000000A
IPbus Packet Data 197: 0x0000000E
IPbus Packet Data 198: 0x536E770A
IPbus Packet Data 199: 0x00007F0E
IPbus Packet Data 200: 0x536E7A0A
IPbus Packet Data 201: 0x00007F0E
IPbus Packet Data 202: 0x0000000A
IPbus Packet Data 203: 0x0000000E
IPbus Packet Data 204: 0x0040080A
IPbus Packet Data 205: 0x0000000E
IPbus Packet Data 206: 0x536E7A0A
IPbus Packet Data 207: 0x00007F0E
IPbus Packet Data 208: 0x00000002
IPbus Packet Data 209: 0x00000006
IPbus Packet Data 210: 0x0000000A
IPbus Packet Data 211: 0x0000000E
```

## References

- SKIROC2: http://omega.in2p3.fr/index.php/products/skiroc.html
  - SKIROC2 specifications: http://omega.in2p3.fr/index.php/download-center/doc_download/248-skiroc2chip.html
  - SKIROC 2 user guide: http://omega.in2p3.fr/index.php/download-center/doc_download/556-skiroc2userguide.html
- IPbus: https://svnweb.cern.ch/trac/cactus/wiki/IPbusFirmware

