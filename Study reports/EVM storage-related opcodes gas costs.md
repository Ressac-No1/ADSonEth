# Study report: EVM storage-related opcodes gas costs

## List of EIPs containing definitions of EVM storage-related opcode gas costs
- EIP-2200: Structured Definitions for Net Gas Metering
- EIP-2929: Gas cost increases for state access opcodes
- EIP-2930: Optional access lists

## Archive of addresses and storage slots accessed
The sets of `accessed_addresses` and `accessed_storage_keys` in the context of a transaction’s execution are maintained to record the ever-visited addresses and their storage slots.

EIP-2929 specifies the start of both sets: `accessed_addresses` as \{`tx.sender`, `tx.to`\} joined with all precompiles, and `accessed_storage_keys` as empty.

EIP-2930 introduces another option to import more addresses and storage slots to the initial value of the two sets. This option supports configuration of preliminary accessed addresses and storage slots, with additional charges for each item: `ACCESS_LIST_ADDRESS_COST` (amount of gas, hereafter) for each address and `ACCESS_LIST_STORAGE_KEY_COST` for each storage slot.

## Storage load (`SLOAD`) gas cost metering
Every `SLOAD` operation checks if the targeted storage slot is recorded in `accessed_storage_keys`; if so, the operation charges `WARM_STORAGE_READ_COST`; otherwise, it charges `COLD_SLOAD_COST` and adds this storage slot into `accessed_storage_keys`.

## Storage update (`SSTORE`) gas cost metering
EIP-2200 describes the detail of `SSTORE` gas cost metering.
- Slot set: the first modification (within the scope of a transaction, hereafter) of an initial zero slot to any non-zero value charges `SSTORE_SET_GAS`;
- Slot activation: the first modification of an initial non-zero slot to any different value charges `SSTORE_RESET_GAS`;
- Slot remodification: every modification of a dirty (already modified) slot charges `SLOAD_GAS`;
- No-op update: every `SSTORE` that does not change the value of a storage slot charges `SLOAD_GAS`;
- Slot clear refund: an initial non-zero slot eventually modified to zero reimburses `SSTORE_CLEARS_SCHEDULE`;
- Slot reset refund: a slot eventually modified to its initial value reimburses `SSTORE_SET_GAS` or `SSTORE_RESET_GAS` (depending on whether its initial value is zero or not) deducted by `SLOAD_GAS`.

## Other opcodes involving address access
Such opcodes include `EXTCODESIZE`, `EXTCODECOPY`, `EXTCODEHASH`, `BALANCE` and all message call operations (`CALL`, `DELEGATECALL`, `STATICCALL` and `CALLCODE`). The proposal in EIP-2929 requires these operations to check if the target address is recorded in `accessed_addresses`, if so, it charges `WARM_STORAGE_READ_COST`, otherwise, it charges `COLD_ACCOUNT_ACCESS_COST` and adds this address into `accessed_addresses`.

## Definition of parameters
According to EIP-2200, EIP-2929 and EIP-2930, all referred gas cost parameters since the Ethereum Berlin Upgrade (https://blog.ethereum.org/2021/03/08/ethereum-berlin-upgrade-announcement) are measured as follows:
- `COLD_SLOAD_COST`: 2100
- `COLD_ACCOUNT_ACCESS_COST`: 2600
- `WARM_STORAGE_READ_COST`: 100
- `SLOAD_GAS`: 100 (the same as `WARM_STORAGE_READ_COST`)
- `SSTORE_SET_GAS`: 20000
- `SSTORE_RESET_GAS`: 2900 (as 5000 - `COLD_SLOAD_COST`)
- `SSTORE_CLEARS_SCHEDULE`: 15000
- `ACCESS_LIST_STORAGE_KEY_COST`: 1900
- `ACCESS_LIST_ADDRESS_COST`: 2400

## Analysis of the measurements
The current gas cost measurement of EVM storage read operations incents address revisiting and storage slot reusing during the execution of a transaction because the first visit of an address or the first access of a storage slot charges the "cold cost", which is significantly more expensive than the "warm cost" for revisiting or reusing. Preliminary configuration, introduced by EIP-2930, can slightly decrease the "cold cost" of some specific addresses and slots by 200 gas each.

As far as storage updates are concerned, the related parameters show that the majority of `SSTORE` gas costs are also caused by the first value updating of any storage slot. In addition, the highest expense of storage-related operations occurs when a slot with zero value is changed to any non-zero value, whereas clearing a non-zero slot to zero is an activity with the highest reimbursement. This is probably due to the dynamicity of the Merkle-Patricia Trie, the data structure on which the Ethereum data storage system is built — all slots with zero value are indeed not stored in the MPT nodes, and the running time and proof size of accessing operations on the MPT are proportional to the depth of the MPT. Therefore, it encourages the limitation of the number of non-zero storage slots.

The major drawback of the current gas cost measurement is the scope restriction — both `SSTORE` and `SLOAD` count their gas costs within a transaction execution. However, it is common for a series of continuous transactions triggering a contract to access and modify almost the same subset of slots of the contract account storage. Even if the Verkle Tree scheme applied to the upcoming upgrade (see EIP-6800: Ethereum state using a unified verkle tree) is barely able to fix the restriction on the scope of transactions. Such a restriction motivates us to believe that it could be a breakthrough for storage-related operation gas savings if further advanced data structure schemes could be designed to eliminate the border of transactions in the address and slot access records or enable rollups of multiple transactions.

