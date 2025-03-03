# TVM Overview

All TON Smart Contracts are executed on their own TON Virtual Machine (TVM). TVM is built on the _stack principle_, which makes it efficient and easy to implement.

This document provides a bird's-eye view of how TVM executes transactions.

:::tip

* There is also a detailed specification — [**Whitepaper**](https://ton.org/tvm.pdf)
* Could be useful — [**TVM C++ implementation**](https://github.com/ton-blockchain/ton/tree/master/crypto/vm)

:::

## Transactions and phases
When some event happens on the account in one of the TON chains, it causes a **transaction**. The most common event is the "arrival of some message", but generally speaking there could be `tick-tock`, `merge`, `split` and other events.

Each transaction consists of up to 5 phases:
1. **Storage phase** - in this phase, storage fees accumulated by the contract due to the occupation of some space in the chain state are calculated. Read more in [Storage Fees](/develop/smart-contracts/fees#storage-fee).
2. **Credit phase** - in this phase, the balance of the contract with respect to a (possible) incoming message value and collected storage fee are calculated.
3. **Compute phase** - in this phase, TVM is executing the contract (see below) and the result of the contract execution is an aggregation of `exit_code`, `actions` (serialized list of actions), `gas_details`, `new_storage` and some others.
4. **Action phase** - if the compute phase was successful, in this phase, `actions` from the compute phase are processed. In particular, actions may include sending messages, updating the smart contract code, updating the libraries, etc. Note that some actions may fail during processing (for instance, if we try to send message with more TON than the contract has), in that case the whole transaction may revert or this action may be skipped (it depends on the mode of the actions, in other words, the contract may send a `send-or-revert` or  `try-send-if-no-ignore` type of message).
5. **Bounce phase** - if the compute phase failed (it returned `exit_code >= 2`), in this phase, the _bounce message_ is formed for the transactions initiated by an incoming message.

## Compute phase
In this phase, the TVM execution occurs.

### TVM state
At any given moment, the TVM state is fully determined by 6 properties:
* Stack (see below)
* Control registers - (see below) to put it simply, this means up to 16 variables which may be directly set and read during execution
* Current continuation - object which describes a currently executed sequence of instructions
* Current codepage - in simple terms, this means the version of TVM which is currently running
* Gas limits - a set of 4 integer values; the current gas limit g<sub>l</sub>, the maximal gas limit g<sub>m</sub>, the remaining gas g<sub>r</sub> and the gas credit g<sub>c</sub>
* Library context - the hashmap of libraries which can be called by TVM 

### TVM is a stack machine
TVM is a last-input-first-output stack machine. In total, there are 7 types of variables which may be stored in stack — three non-cell types:
* Integer - signed 257-bit integers
* Tuple - ordered collection of up to 255 elements having arbitrary value types, possibly distinct.
* Null

And four distinct flavours of cells:
* Cell - basic (possibly nested) opaque structure used by TON Blockchain for storing all data
* Slice - a special object which allows you to read from a cell
* Builder - a special object which allows you to create new cells
* Continuation - a special object which allows you to use a cell as source of TVM instructions

### Control registers
* `c0` — Contains the next continuation or return continuation (similar to the subroutine return address in conventional designs). This value must be a Continuation.
* `c1` — Contains the alternative (return) continuation; this value must be a Continuation. 
* `c2` — Contains the exception handler. This value is a Continuation, invoked whenever an exception is triggered.
* `c3` — Supporting register, contains the current dictionary, essentially a hashmap containing the code of all functions used in the program. This value must be a Continuation. 
* `c4` — Contains the root of persistent data, or simply the `data` section of the contract. This value is a Cell.
* `c5` — Contains the output actions. This value is a Cell.
* `c7` — Contains the root of temporary data. It is a Tuple.

### Initialization of TVM

TVM initializes when transaction execution gets to the Computation phase, and then executes commands (opcodes) from _Current continuation_ until there are no more commands to execute (and no continuation for return jumps).

Detailed description of the initialization process can be found here: [TVM Initialization](/learn/tvm-instructions/tvm-initialization.md)

## TVM instructions

The list of TVM instructions can be found here: [TVM instructions](/learn/tvm-instructions/instructions).

### Result of TVM execution
Besides exit_code and consumed gas data, TVM indirectly outputs the following data:
* c4 register - the cell which will be stored as new `data` of the smart-contract (if execution will not be reverted on this or later phases)
* c5 register - (list of output actions) the cell with last action in the list and reference to the cell with prev action (recursively)

All other register values will be neglected.

Note, that since there is a limit on max cell-depth `<1024`, and particularly the limit on c4 and c5 depth `<=512`, there will be a limit on the number of output actions in one tx `<=255`. If a contract needs to send more than that, it may send a message with the request `continue_sending` to itself and send all necessary messages in subsequent transactions.
