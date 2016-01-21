# transport-layer

## Introduction

According to the Fermilab Memo on Dec 14, 2015, the transport layer protocol is loosely based on the [IPbus protocol](ipbus_protocol_v2_0.pdf) (v2.0, Draft 9, Dec 2013), but with some simplifications. The data is packed into 32-bit words, with the following features:

1. We do not use the IPbus packet header. We send IPbus control packets directly.
1. An IPbus control packet consists of one command word and a number of data words, with a minimum of 0 and a maximum of 255.
1. There are two types of control packet: request and response.
1. The command word is a modified version of the IPbus transaction header except the "Transaction ID" field is replaced with an "Address" field.

  ```
  | 31    28 | 27              16 | 15               8 | 7      4 | 3      0 |
  |----------|--------------------|--------------------|----------|----------|
  | Protocol | Starting           | Number of          | Type ID  | Info code|
  | version  | address            | words              |          |          |
  | (4 bits) | (12 bits)          | (8 bits)           | (4 bits) | (4 bits) |
  ```

