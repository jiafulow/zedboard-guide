# transport-layer

## Introduction

According to the Fermilab Memo on Dec 14, 2015, we use the transport layer protocol that is loosely based on the [IPbus protocol](ipbus_protocol_v2_0.pdf) (v2.0, Draft 9, Dec 2013) with some simplifications. The data is always packed into 32-bit words, with the following features:

1. We do not use an IPbus packet header. We send IPbus control packets directly.
1. An IPbus control packet consists of one command word and a number of data words, with a minimum of 0 and a maximum of 255. The command word specifies the type of transaction. For each type of transaction, there are a request packet and a response packet.
1. The command word is a modified version of the IPbus transaction header except the "Transaction ID" field is replaced with an "Starting Address" field.

  ```
  Our implementation:
  
  +--------------------------------------------------------------------------+
  | 31    28 | 27              16 | 15               8 | 7      4 | 3      0 |
  |----------|--------------------|--------------------|----------|----------|
  | Protocol | Starting           | Number of          | Type ID  | Info Code|
  | Version  | Address            | Words              |          |          |
  | (4 bits) | (12 bits)          | (8 bits)           | (4 bits) | (4 bits) |
  +--------------------------------------------------------------------------+

  Compared to IPbus v2.0 implementation:

  +--------------------------------------------------------------------------+
  | 31    28 | 27              16 | 15               8 | 7      4 | 3      0 |
  |----------|--------------------|--------------------|----------|----------|
  | Protocol | Transaction ID     | Number of          | Type ID  | Info Code|
  | Version  |                    | Words              |          |          |
  | (4 bits) | (12 bits)          | (8 bits)           | (4 bits) | (4 bits) |
  +--------------------------------------------------------------------------+

  ```

  - `Protocol Version`: reserved, currently 0.
  - `Starting Address`: the intended address for this transaction. The address auto-increments by 4 with each word (since 32 bits = 4 bytes).
  - `Number of Words`: the length of this transaction, excluding the command word. It is a 8-bit number with a min of 0 and a max of 255.
  - `Type ID`: the type of this transaction. It is either Read (Type ID = 0x0) or Write (Type ID = 0x1).
  - `Info Code`: the direction and error state of the transaction. A request has Info Code = 0xF; a successful response has Info Code = 0x0. All other values mean errors during the transaction.

1. The command word is followed immediately by the variable-length sequence of data words.

1. We use little-endian for the command word and the data words (default on Linux systems).

1. We anticipate that a range of addreses will be assigned to a block RAM, which may contain the digitzed data from an event, or the clock sequencer data, or they may be individual addresses -- such as a register that counts the number of triggers, or even an address that has no data associated with it: for example, an address that initiates a software trigger.

1. All transactions are initialized by the ZedBoard and all transactions require a response from the FMC_IO. The rely starts as soon as data for the reply is available.

1. An example **Write** transaction:

  ```
  **Request**

         +--------------------------------------------------------------------------+
         | 31    28 | 27              16 | 15               8 | 7      4 | 3      0 |
         |----------|--------------------|--------------------|----------|----------|
  Word 0 | Ver = 0  | Starting Address   | Num of words = N   | Type=0x1 | Info=0xF |
         |--------------------------------------------------------------------------|
  Word 1 | Data to be written to Starting Address                                   |
         |--------------------------------------------------------------------------|
  Word 2 | Data to be written to Starting Address+4                                 |
         |--------------------------------------------------------------------------|
    ...  |                                                                          |
         |--------------------------------------------------------------------------|
  Word N | Data to be written to Starting Address+4(N-1)                            |
         +--------------------------------------------------------------------------+

  **Response**

         +--------------------------------------------------------------------------+
         | 31    28 | 27              16 | 15               8 | 7      4 | 3      0 |
         |----------|--------------------|--------------------|----------|----------|
  Word 0 | Ver = 0  | Starting Address   | Num of words = N   | Type=0x1 | Info=0x0 |
         +--------------------------------------------------------------------------+
  ```

1. An example **Read** transaction:

  ```
  **Request**

         +--------------------------------------------------------------------------+
         | 31    28 | 27              16 | 15               8 | 7      4 | 3      0 |
         |----------|--------------------|--------------------|----------|----------|
  Word 0 | Ver = 0  | Starting Address   | Num of words = N   | Type=0x0 | Info=0xF |
         +--------------------------------------------------------------------------+

  **Response**

         +--------------------------------------------------------------------------+
         | 31    28 | 27              16 | 15               8 | 7      4 | 3      0 |
         |----------|--------------------|--------------------|----------|----------|
  Word 0 | Ver = 0  | Starting Address   | Num of words = N   | Type=0x0 | Info=0x0 |
         +--------------------------------------------------------------------------+
  Word 1 | Data read from Starting Address                                          |
         +--------------------------------------------------------------------------+
  Word 2 | Data read from Starting Address+4                                        |
         +--------------------------------------------------------------------------+
    ...  |                                                                          |
         +--------------------------------------------------------------------------+
  Word N | Data read from Starting Address+4(N-1)                                   |
         +--------------------------------------------------------------------------+

  ```

### Software Design

The software has been uploaded to [test_beam_DAQ_2015](https://github.com/jbueghly/test_beam_DAQ_2015/) GitHub repository.
The software can be found in the directory [projects/testbeam_6a/software/sk2_sc_transport](https://github.com/jbueghly/test_beam_DAQ_2015/tree/master/projects/testbeam_6a/software/sk2_sc_transport).

The implementations can be found in `IPbusLite.h` and `IPbusLite.c`. The function prototypes are:

```c
// Create IPbusLite request transaction
int create_ipbus_txn(const Command * cmnd, Transaction * txn);
uint32_t create_ipbus_txn_header(uint32_t addr, uint16_t nwords, uint8_t type, uint8_t info);
uint32_t create_ipbus_txn_data(uint32_t word);

// Extract IPbusLite response transaction
int extract_ipbus_txn(const Transaction * txn, Command * cmnd);
void extract_ipbus_txn_header(uint32_t header, uint32_t* addr, uint16_t* nwords, uint8_t* type, uint8_t* info);
void extract_ipbus_txn_data(uint32_t data, uint32_t* word);
```

where `Command` and `Transaction` are defined as
```c
struct Command
{
    uint32_t addr;
    uint16_t nwords;
    uint8_t type;
    uint8_t info;
    uint32_t data[255];  // maximum is 255 data words
};

struct Transaction
{
    uint32_t data[256];  // maximum is 1 header word + 255 data words
};
```

Basically, `Command` is the transaction before it is packed; while `Transaction` is the transaction after. `create_ipbus_txn()` and `extract_ipbus_txn()` convert `Command` into `Transaction` and vice versa.

### Program Usage

- `sk2_sc_ipbus_write_txn`

```shell
usage: sk2_sc_write_txn N [WORD 1, ..., WORD N]

       N is in decimal, WORD is in HEX, e.g.
       sk2_sc_write_txn 0
       sk2_sc_write_txn 4 0x12 0x34 0x99 0xFF

# Write 0 word
$ sk2_sc_write_txn 0
Request:
  addr=0xDEADBEEF, nwords=0, words=
Dump txn:
0x0EEF001F

Response:
  addr=0x00000EEF, nwords=0
Dump txn:
0x0EEF0010

# Write 4 words
$ sk2_sc_write_txn 4 0x12 0x34 0x99 0xFF
Request:
  addr=0xDEADBEEF, nwords=4, words=
  0x00000012 0x00000034 0x00000099 0x000000FF 
Dump txn:
0x0EEF041F
0x00000012
0x00000034
0x00000099
0x000000FF

Response:
  addr=0x00000EEF, nwords=4
Dump txn:
0x0EEF0410
```

- `sk2_sc_ipbus_read_txn`

```shell
usage: sk2_sc_read_txn N

       N is in decimal, e.g.
       sk2_sc_read_txn 4

# Read 0 word
$ sk2_sc_read_txn 0
Request:
  addr=0xDEADBEEF, nwords=0
Dump txn:
0x0EEF000F

Response:
  addr=0x00000EEF, nwords=0, words=
Dump txn:
0x0EEF0000

# Read 4 words (currently it reads fake, ordered data)
$ sk2_sc_read_txn 4
Request:
  addr=0xDEADBEEF, nwords=4
Dump txn:
0x0EEF040F

Response:
  addr=0x00000EEF, nwords=4, words=
  0x00000000 0x00000001 0x00000002 0x00000003 
Dump txn:
0x0EEF0400
0x00000000
0x00000001
0x00000002
0x00000003

```

