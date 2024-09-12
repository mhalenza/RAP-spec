# RAP - [Renote] Register Access Protocol

RAP is a protocol specification for remotely accessing registers in a device over a variety of byte-oriented transports.

NOTE: This specification is still being revised, DO NOT use it for production designs yet.

## Table of Contents
- [Configuration](#configuration)
    - [Sizing](#sizing)
    - [Features](#features)
    - [Transport](#transport)
- [Transports](#transports)
- [Messages](#messages)

# Configuration
RAP has a number of configurable knobs that change details about the protocol without changing the overall structure.
RAP naming has the form "RAP-<Sizing>+<Features>/<Transport>".

## Sizing
Sizing involves 3 major aspects of the protocol: Address Size, Data Size, and CRC Size.  
Naming is `A<x>D<y>[C<z>]L<w>` where
- `<x>`, aka "Address Size", is an integer in the range 1 to 64 (inclusive). It represents the number of valid bits in an Address field.
- `<y>`, aka "Data Size", is an integer in the range 1 to 64 (inclusive).  It represents the number of bits in a Data field.
- `<z>`, aka "CRC Bytes", is an integer from {1, 2, 4, 8}. It represents the number of bytes in a CRC field.  The whole "C<z>" part is also optional and when it is left off, it implies that messages do not carry a CRC field.
- `<w>`, aka "Length Bytes", is an integer from 1 to 4.  It represents the number of bytes in a Length field.  Required when the S, F, I, or C feature flags are present.

There is a special note about Address Size and Data Size:
The Protocol Name contains the number of *valid bits* but in the messages Address and Data fields are rounded up to the next multiple of 8.
This is to keep the messages byte-oriented without bit-packing

Here's some examples:
| Protocol Name | Address Bits | Address Bytes | Data Bits | Data Bytes | CRC Bytes |
| :------- | :-: | :-: | :-: | :-: | :-: |
| A32D32C16 | 32 |  4 | 32 |  4 |  2 |
|  A12D32C8 | 12 |  2 | 32 |  4 |  8 |
|      A8D9 |  8 |  1 |  9 |  2 |  0 |

## Features
The "Features" part of the name is a list of single characters, the presence of which indicates that the implementation included optional features.  
Defined features are:
- `S` - includes support for Sequential type messages, with a fixed increment that is equal to the number of Data Bytes
- `F` - includes support for FIFO type messages  
    `F` is exactly equivalent to Sequential type messages where the `increment` is zero
- `I` - includes support for Sequential type messages, with arbitrary `increment` values
- `C` - includes support for Compressed type messages
- `Q` - includes support for asynchronous/interrupt messages
- `M` - includes support for ReadModifyWrite type messages

## Transport
The "Transport" part of the name indicates which transport is used.  
The defined transports are:
- `UART` - framed messages over a UART type byte stream
- `SpW` - messages encapsulated in SpW packets
- `UDP` - messages encapsulated in UDP packets and sent over an IPv4 or IPv6 network  
    Whether IPv4 OR IPv6 OR Both is supported is defined by the implementation.
- `TCP` - framed messages sent over a TCP stream on top of an IPv4 or IPv6 network  
    Whether IPv4 OR IPv6 OR Both is supported is defined by the implementation.
- `Ethernet` - messages encapsulated in raw Ethernet frames

Generally, other transports can be accomodated easily.
The major distinction is whether the underlying transport can "frame" messages or not.  
UDP, SpW, and Ethernet are examples of transports that can frame messages because messages must fit within a single UDP Packet, SpW Packet, or Ethernet Frame.  
UART and TCP are examples of "stream" transports which cannot (on their own) frame messages.  
For "stream" oriented transports, RAP must include some mechanism to indicate message boundaries - see the Framing section for more info.

# Transports
| Transport | Framed / Stream | Max Message Size | 
| :-------: | :-------------: | :--------------: |
| UART | Stream | Unlimited* |
| SpW | Framed | Unlimited* |
| Ethernet/IPv4/UDP | Framed | 1472 Bytes |
| Ethernet/IPv6/UDP | Framed | 1454 Bytes |
| Ethernet/IPv4/TCP | Stream | Unlimited* |
| Ethernet/IPv6/TCP | Stream | Unlimited* |
| Ethernet | Framed | 1500 Bytes |

## Max Message Size
The Maximum Message Size (in bytes) is generally dictated by the transport and to a lesser extent the implementation.

### Stream Transports
For stream oriented transports (UART, TCP) the maximum message size is "unlimited" in that the transport doesn't really limit the message size.
However, practical implementations will have to buffer messages before they can be processed, thereby imposing a limit.

### Framed Transports
Framed transports generally have a maximum "datagram" size imposed by their respective standards.

#### Ethernet, IPv4/IPv6, UDP
A number of factors are involved:
- Ethernet imposes a maximum frame size of 1522 bytes.
- Ethernet has a header/tail overhead of 22 bytes.
- IPv4 has a header overhead of 20 bytes.
- IPv6 has a header overhead of 40 bytes.
- UDP has a header overhead of 8 bytes.

Therefore, "Ethernet/IPv4/UDP" has a Max Message Size of 1472 bytes and "Ethernet/IPv6/UDP" has a Max Message Size of 1454 bytes.

Other stackups involving UDP are possible and the maximum message size should be calculated accordingly.

# Messages
All messages start with a "Transaction ID" byte followed by a "Message Type" byte and end with a 0, 1, 2, or 4 byte CRC.
The payload bytes in between are defined by the Message Type value.

## Transaction ID
The Transaction ID byte is a transaction identifier.
Clients pick an ID and Server's reflect that ID back in responses (and do nothing else with it).
Clients that do not support interleaving may choose to use a constant value for the ID, but implementations wishing to support interleaved transactions must choose unique IDs for each transaction.

## Message Type Byte
| Bit | Meaning |
| :-: | :------ |
|  7  | 0=Command, 1=Response |
|  6  | **Cmd:** 0=Non-Posted, 1=Posted <br/> **Resp:** 0=ACK, 1=NAK |
| 5:4 | 0=Read, 1=Write, 2=ReadModifyWrite, 3=Interrupt *(Resp Only)* |
| 3:2 | 0=Single, 1=Seq/Fifo, 2=Compressed, 3=RESERVED |
|  1  | 0 *(RESERVED)* |
|  0  | Odd Parity |

### Non-Posted vs Posted
Non-Posted commands expect a response, as either an ACK or a NAK.
Posted commands do not get responses.
As a result, Posted can only be used with Writes and is not valid with Reads.

### ReadModifyWrite
In a single command, performs a read-modify-write operation on the register value.
If the supplied

### Interrupt
Requires the `Q` feature flag.
Valid only for ACK Responses.
Informs the Host that some event needs servicing.

### Seq/Fifo
Operates on one or more sequential registers per command.
The command body contains an `increment` field which indicates the step size for each successive data element.

The valid values for the `increment` depends on what feature flags the implementation supports:
| Flag | Valid Values | Note |
| :-: | :- | :- |
| `S` | `DS` | Sequential-only, aligned to the Data Size in bytes. |
| `F` | 0 | FIFO mode. |
| `I` | Any value from 0 to 255 | Any increment allowed, super flexible! |

Note that any combination of the 3 flags is allowed.

### Compressed
Requires feature flag `C`.
Reads or writes multiple randomly-addressed registers in a single command.

## Valid Message Types
| CMD | Description |
| :-: | :---------- |
| `01` | Command, Non-Posted, Read, Single |
| `80` | Response, ACK, Read, Single |
| `C1` | Response, NAK, Read, Single |
| `10` | Command, Non-Posted, Write, Single |
| `51` | Command, Posted, Write, Single |
| `91` | Response, ACK, Write, Single |
| `D0` | Response, NAK, Write, Single |
| `04` | Command, Non-Posted, Read, Seq/Fifo |
| `85` | Response, ACK, Read, Seq/Fifo |
| `C4` | Response, NAK, Read, Seq/Fifo |
| `15` | Command, Non-Posted, Write, Seq/Fifo |
| `54` | Command, Posted, Write, Seq/Fifo |
| `94` | Response, ACK, Write, Seq/Fifo |
| `D5` | Response, NAK, Write, Seq/Fifo |
| `08` | Command, Non-Posted, Read, Compressed |
| `89` | Response, ACK, Read, Compressed |
| `C8` | Response, NAK, Read, Compressed |
| `19` | Command, Non-Posted, Write, Compressed |
| `58` | Command, Posted, Write, Compressed |
| `98` | Response, ACK, Write, Compressed |
| `D9` | Response, NAK, Write, Compressed |
| `20` | Command, Non-Posted, ReadModifyWrite, Single |
| `61` | Command, Posted, ReadModifyWrite, Single |
| `A1` | Response, ACK, ReadModifyWrite, Single |
| `E0` | Response, NAK, ReadModifyWrite, Single |
| `B0` | Response, ACK, Interrupt, Single |

## Command Messages

### Single Read
Performs a read of a single register.

Required Feature Flags: none  
Message Size: `2 + AS + CS`

| Field Name | Size (Bytes) | Use |
| :-: | :-: | :- |
| Transaction ID | 1 | Transaction Identifier |
| `01` | 1 | Message Type |
| Address | `AS` | The address to read from |
| CRC | `CS` | Message CRC |

### Single Write
Performs a write of a single regsiter

Required Feature Flags: none  
Message Size: `2 + AS + DS + CS`

| Field Name | Size (Bytes) | Use |
| :-: | :-: | :- |
| Transaction ID | 1 | Transaction Identifier |
| `10` / `51` | 1 | Message Type |
| Address | `AS` | The address to write to |
| Data | `DS` | The data to write |
| CRC | `CS` | Message CRC |

### Seq/Fifo Read
Performs multiple reads of one or more registers.

Required Feature Flags: S, F, and/or I  
Message Size: `2 + AS + 2*LS + CS`

| Field Name | Size (Bytes) | Use |
| :-: | :-: | :- |
| Transaction ID | 1 | Transaction Identifier |
| `04` | 1 | Message Type |
| Address | `AS` | The first address to read from |
| Increment | `LS` | How much to increment the address between iterations |
| Length | `LS` | The number of reads to perform |
| CRC | `CS` | Message CRC |

### Seq/Fifo Write
Performs multiple writes of one or more registers.

Required Feature Flags: S, F, and/or I  
Message Size: `2 + AS + LS + L*DS + CS`

| Field Name | Size (Bytes) | Use |
| :-: | :-: | :- |
| Transaction ID | 1 | Transaction Identifier |
| `15` / `54` | 1 | Message Type |
| Address | `AS` | The first address to write to |
| Increment | `LS` | How much to increment the address between iterations |
| Data[] | `L*DS` | `L` Data elements holding the data to write |
| CRC | `CS` | Message CRC |

### Compressed Read
Performs multiple reads of non-sequential registers.

Required Feature Flags: C  
Message Size: `2 + L*AS + CS`

| Field Name | Size (Bytes) | Use |
| :-: | :-: | :- |
| Transaction ID | 1 | Transaction Identifier |
| `08` | 1 | Message Type |
| Address[] | `L*AS` | `L` Address elements indicaiting which registers to read |
| CRC | `CS` | Message CRC |

### Compressed Write
Performs multiple writes of non-sequential registers.

Required Feature Flags: C  
Message Size: `2 + L*(AS + DS) + CS`

| Field Name | Size (Bytes) | Use |
| :-: | :-: | :- |
| Transaction ID | 1 | Transaction Identifier |
| `19` / `58` | 1 | Message Type |
| Address/Data[] | `L*(AS + DS)` | `L` address & data pairs to write |
| CRC | `CS` | Message CRC |

### Single ReadModifyWrite
Performs a read-modify-write operation on a single register.

The `Mask` field indicates which bits should be cleared and then `Data` is bitwise-or'd into the register.
It is undefined what happens if `Data` contains bits that are set which correspond to zero bits in the `Mask`.  ie, it is up to the implementation to decide whether to bitwise-AND `Data` and `Mask` before performing the bitwise-or.
Therefore, it would be prudent for client side implementations to perform this masking before sending the command.

Required Feature Flags: M  
Message Size: `2 + AS + 2*DS + CS`

| Field Name | Size (Bytes) | Use |
| :-: | :-: | :- |
| Transaction ID | 1 | Transaction Identifier |
| `20` / `61` | 1 | Message Type |
| Address | `AS` | The address to modify |
| Mask | `DS` | The mask to apply to the field as well as the data |
| Data | `DS` | The data to bitwise-or into the field |
| CRC | `CS` | Message CRC |

## Response Messages
