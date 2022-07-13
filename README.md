<img src="https://raw.githubusercontent.com/playmint/lallafa/main/lallafa.jfif" width="300" height="300"/>

[![NPM Package](https://img.shields.io/npm/v/@playmint/lallafa.svg?style=flat-square)](https://www.npmjs.com/package/@playmint/lallafa)
---
# Lallafa
An EVM gas profiling tool written in Typescript.

## Installation
`npm install --save-dev @playmint/lallafa`

## How it works
Lallafa produces a gas usage report by ingesting a debug trace, along with some compiler input and output. It outputs profiling data which is collated at both the instruction level, and the line-of-code level.

## Example
Check out the [test](https://github.com/playmint/lallafa/tree/main/test) project for detailed examples.

Here's a way to profile a simple contract
Storage.sol:
```solidity
//SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.0;

contract Storage {
    uint256 _value;

    function setValue(uint256 value) public {
        _value = value;
    }
}
```
profileStorage.ts:
```ts
import hre from "hardhat";
import { Storage__factory } from "../typechain-types";
import fs from "fs";
import { profile, sourcesProfileToString, instructionsProfileToString, ContractInfoMap } from "../../src";


async function main() {
    const storageFactory = new Storage__factory((await hre.ethers.getSigners())[0]);
    const storage = await storageFactory.deploy();
    const tx = await (await storage.setValue(42)).wait();
    const debugTrace = await hre.ethers.provider.send("debug_traceTransaction",
        [tx.transactionHash, {
            "disableStack": true,
            "disableMemory": true,
            "disableStorage": true
        }]);

    const buildInfo = await hre.artifacts.getBuildInfo("contracts/Storage.sol:Storage");
    if (!buildInfo) {
        throw new Error("couldn't find build info");
    }
    const output = buildInfo.output.contracts["contracts/Storage.sol"]["Storage"];

    const contracts: ContractInfoMap = {}
    contracts[storage.address] = {
        output: {
            bytecode: {
                bytecode: output.evm.bytecode.object,
                sourceMap: output.evm.bytecode.sourceMap,
                generatedSources: (output.evm.bytecode as any).generatedSources || []
            },
            deployedBytecode: {
                bytecode: output.evm.deployedBytecode.object,
                sourceMap: output.evm.deployedBytecode.sourceMap,
                generatedSources: (output.evm.deployedBytecode as any).generatedSources || []
            },
            sources: buildInfo.output.sources
        },
        input: buildInfo.input
    };

    const result = profile(
        debugTrace,
        false, // isDeployment
        storage.address,
        contracts);

    fs.writeFileSync("sources_profile.txt", sourcesProfileToString(result));
    fs.writeFileSync("instruction_profile.txt", instructionsProfileToString(result));
}

main().catch(e => console.error(e));
```
This script outputs profiles of both the source code and the asm instructions to show what these both look like.
sources_profile.txt:
```
// 5FBDB2315678AFECB367F032D93F642F64180AA3
============================================

// contracts/Storage.sol

Gas   |
---------------------------------------------------------------------------------------------------------------------------------------
97    | //SPDX-License-Identifier: UNLICENSED
0     | pragma solidity ^0.8.0;
0     | 
0     | contract Storage {
0     |     uint256 _value;
0     | 
68    |     function setValue(uint256 value) public {
22114 |         _value = value;
0     |     }
0     | }
0     | 

// #utility.yul

Gas   |
---------------------------------------------------------------------------------------------------------------------------------------
0     | {
0     | 
0     |     function allocate_unbounded() -> memPtr {
0     |         memPtr := mload(64)
0     |     }
0     | 
0     |     function revert_error_dbdddcbe895c83990c08b3492a0e83918d802a52331272ac6fdb6a7c4aea3b1b() {
0     |         revert(0, 0)
0     |     }
0     | 
0     |     function revert_error_c1322bf8034eace5e0b5c7295db60986aa89aae5e0ea0873e4689e076861a5db() {
0     |         revert(0, 0)
0     |     }
0     | 
20    |     function cleanup_t_uint256(value) -> cleaned {
8     |         cleaned := value
0     |     }
0     | 
11    |     function validator_revert_t_uint256(value) {
38    |         if iszero(eq(value, cleanup_t_uint256(value))) { revert(0, 0) }
0     |     }
0     | 
22    |     function abi_decode_t_uint256(offset, end) -> value {
11    |         value := calldataload(offset)
18    |         validator_revert_t_uint256(value)
0     |     }
0     | 
22    |     function abi_decode_tuple_t_uint256(headStart, dataEnd) -> value0 {
32    |         if slt(sub(dataEnd, headStart), 32) { revert_error_dbdddcbe895c83990c08b3492a0e83918d802a52331272ac6fdb6a7c4aea3b1b() }
0     | 
2     |         {
0     | 
3     |             let offset := 0
0     | 
32    |             value0 := abi_decode_t_uint256(add(headStart, offset), dataEnd)
0     |         }
0     | 
0     |     }
0     | 
0     | }
0     | 


```
instructions_profile.txt:
```
// 5FBDB2315678AFECB367F032D93F642F64180AA3
============================================

Gas   | PC | Asm                                                                                          | Bytecode
------------------------------------------------------------------------------------------------------------------------
3     | 0  | PUSH1 0x80                                                                                   | 0x6080
3     | 2  | PUSH1 0x40                                                                                   | 0x6040
12    | 4  | MSTORE                                                                                       | 0x52
2     | 5  | CALLVALUE                                                                                    | 0x34
3     | 6  | DUP1                                                                                         | 0x80
3     | 7  | ISZERO                                                                                       | 0x15
3     | 8  | PUSH1 0xF                                                                                    | 0x600F
10    | A  | JUMPI [in]                                                                                   | 0x57
0     | B  | PUSH1 0x0                                                                                    | 0x6000
0     | D  | DUP1                                                                                         | 0x80
0     | E  | REVERT                                                                                       | 0xFD
1     | F  | JUMPDEST                                                                                     | 0x5B
2     | 10 | POP                                                                                          | 0x50
3     | 11 | PUSH1 0x4                                                                                    | 0x6004
2     | 13 | CALLDATASIZE                                                                                 | 0x36
3     | 14 | LT                                                                                           | 0x10
3     | 15 | PUSH1 0x28                                                                                   | 0x6028
10    | 17 | JUMPI [in]                                                                                   | 0x57
3     | 18 | PUSH1 0x0                                                                                    | 0x6000
3     | 1A | CALLDATALOAD                                                                                 | 0x35
3     | 1B | PUSH1 0xE0                                                                                   | 0x60E0
3     | 1D | SHR                                                                                          | 0x1C
3     | 1E | DUP1                                                                                         | 0x80
3     | 1F | PUSH4 0x55241077                                                                             | 0x6355241077
3     | 24 | EQ                                                                                           | 0x14
3     | 25 | PUSH1 0x2D [setValue_uint256_0]                                                              | 0x602D
10    | 27 | JUMPI [in]                                                                                   | 0x57
0     | 28 | JUMPDEST                                                                                     | 0x5B
0     | 29 | PUSH1 0x0                                                                                    | 0x6000
0     | 2B | DUP1                                                                                         | 0x80
0     | 2C | REVERT                                                                                       | 0xFD
1     | 2D | JUMPDEST [setValue_uint256_0]                                                                | 0x5B
3     | 2E | PUSH1 0x43                                                                                   | 0x6043
3     | 30 | PUSH1 0x4                                                                                    | 0x6004
3     | 32 | DUP1                                                                                         | 0x80
2     | 33 | CALLDATASIZE                                                                                 | 0x36
3     | 34 | SUB                                                                                          | 0x03
3     | 35 | DUP2                                                                                         | 0x81
3     | 36 | ADD                                                                                          | 0x01
3     | 37 | SWAP1                                                                                        | 0x90
3     | 38 | PUSH1 0x3F                                                                                   | 0x603F
3     | 3A | SWAP2                                                                                        | 0x91
3     | 3B | SWAP1                                                                                        | 0x90
3     | 3C | PUSH1 0x85 [abi_decode_tuple_t_uint256_0]                                                    | 0x6085
8     | 3E | JUMP [in]                                                                                    | 0x56
1     | 3F | JUMPDEST [setValue_uint256_1]                                                                | 0x5B
3     | 40 | PUSH1 0x45 [setValue_uint256_3]                                                              | 0x6045
8     | 42 | JUMP [in]                                                                                    | 0x56
1     | 43 | JUMPDEST [setValue_uint256_2]                                                                | 0x5B
0     | 44 | STOP                                                                                         | 0x00
1     | 45 | JUMPDEST [setValue_uint256_3]                                                                | 0x5B
3     | 46 | DUP1                                                                                         | 0x80
3     | 47 | PUSH1 0x0                                                                                    | 0x6000
3     | 49 | DUP2                                                                                         | 0x81
3     | 4A | SWAP1                                                                                        | 0x90
22100 | 4B | SSTORE                                                                                       | 0x55
2     | 4C | POP                                                                                          | 0x50
2     | 4D | POP                                                                                          | 0x50
8     | 4E | JUMP [out]                                                                                   | 0x56
0     | 4F | JUMPDEST [revert_error_dbdddcbe895c83990c08b3492a0e83918d802a52331272ac6fdb6a7c4aea3b1b_0]   | 0x5B
0     | 50 | PUSH1 0x0                                                                                    | 0x6000
0     | 52 | DUP1                                                                                         | 0x80
0     | 53 | REVERT                                                                                       | 0xFD
1     | 54 | JUMPDEST [cleanup_t_uint256_0]                                                               | 0x5B
3     | 55 | PUSH1 0x0                                                                                    | 0x6000
3     | 57 | DUP2                                                                                         | 0x81
3     | 58 | SWAP1                                                                                        | 0x90
2     | 59 | POP                                                                                          | 0x50
3     | 5A | SWAP2                                                                                        | 0x91
3     | 5B | SWAP1                                                                                        | 0x90
2     | 5C | POP                                                                                          | 0x50
8     | 5D | JUMP [out]                                                                                   | 0x56
1     | 5E | JUMPDEST [validator_revert_t_uint256_0]                                                      | 0x5B
3     | 5F | PUSH1 0x65                                                                                   | 0x6065
3     | 61 | DUP2                                                                                         | 0x81
3     | 62 | PUSH1 0x54 [cleanup_t_uint256_0]                                                             | 0x6054
8     | 64 | JUMP [in]                                                                                    | 0x56
1     | 65 | JUMPDEST [validator_revert_t_uint256_1]                                                      | 0x5B
3     | 66 | DUP2                                                                                         | 0x81
3     | 67 | EQ                                                                                           | 0x14
3     | 68 | PUSH1 0x6F [validator_revert_t_uint256_2]                                                    | 0x606F
10    | 6A | JUMPI [in]                                                                                   | 0x57
0     | 6B | PUSH1 0x0                                                                                    | 0x6000
0     | 6D | DUP1                                                                                         | 0x80
0     | 6E | REVERT                                                                                       | 0xFD
1     | 6F | JUMPDEST [validator_revert_t_uint256_2]                                                      | 0x5B
2     | 70 | POP                                                                                          | 0x50
8     | 71 | JUMP [out]                                                                                   | 0x56
1     | 72 | JUMPDEST [abi_decode_t_uint256_0]                                                            | 0x5B
3     | 73 | PUSH1 0x0                                                                                    | 0x6000
3     | 75 | DUP2                                                                                         | 0x81
3     | 76 | CALLDATALOAD                                                                                 | 0x35
3     | 77 | SWAP1                                                                                        | 0x90
2     | 78 | POP                                                                                          | 0x50
3     | 79 | PUSH1 0x7F                                                                                   | 0x607F
3     | 7B | DUP2                                                                                         | 0x81
3     | 7C | PUSH1 0x5E [validator_revert_t_uint256_0]                                                    | 0x605E
8     | 7E | JUMP [in]                                                                                    | 0x56
1     | 7F | JUMPDEST [abi_decode_t_uint256_1]                                                            | 0x5B
3     | 80 | SWAP3                                                                                        | 0x92
3     | 81 | SWAP2                                                                                        | 0x91
2     | 82 | POP                                                                                          | 0x50
2     | 83 | POP                                                                                          | 0x50
8     | 84 | JUMP [out]                                                                                   | 0x56
1     | 85 | JUMPDEST [abi_decode_tuple_t_uint256_0]                                                      | 0x5B
3     | 86 | PUSH1 0x0                                                                                    | 0x6000
3     | 88 | PUSH1 0x20                                                                                   | 0x6020
3     | 8A | DUP3                                                                                         | 0x82
3     | 8B | DUP5                                                                                         | 0x84
3     | 8C | SUB                                                                                          | 0x03
3     | 8D | SLT                                                                                          | 0x12
3     | 8E | ISZERO                                                                                       | 0x15
3     | 8F | PUSH1 0x98 [abi_decode_tuple_t_uint256_2]                                                    | 0x6098
10    | 91 | JUMPI [in]                                                                                   | 0x57
0     | 92 | PUSH1 0x97                                                                                   | 0x6097
0     | 94 | PUSH1 0x4F [revert_error_dbdddcbe895c83990c08b3492a0e83918d802a52331272ac6fdb6a7c4aea3b1b_0] | 0x604F
0     | 96 | JUMP [in]                                                                                    | 0x56
0     | 97 | JUMPDEST [abi_decode_tuple_t_uint256_1]                                                      | 0x5B
1     | 98 | JUMPDEST [abi_decode_tuple_t_uint256_2]                                                      | 0x5B
3     | 99 | PUSH1 0x0                                                                                    | 0x6000
3     | 9B | PUSH1 0xA4                                                                                   | 0x60A4
3     | 9D | DUP5                                                                                         | 0x84
3     | 9E | DUP3                                                                                         | 0x82
3     | 9F | DUP6                                                                                         | 0x85
3     | A0 | ADD                                                                                          | 0x01
3     | A1 | PUSH1 0x72 [abi_decode_t_uint256_0]                                                          | 0x6072
8     | A3 | JUMP [in]                                                                                    | 0x56
1     | A4 | JUMPDEST [abi_decode_tuple_t_uint256_3]                                                      | 0x5B
3     | A5 | SWAP2                                                                                        | 0x91
2     | A6 | POP                                                                                          | 0x50
2     | A7 | POP                                                                                          | 0x50
3     | A8 | SWAP3                                                                                        | 0x92
3     | A9 | SWAP2                                                                                        | 0x91
2     | AA | POP                                                                                          | 0x50
2     | AB | POP                                                                                          | 0x50
8     | AC | JUMP [out]                                                                                   | 0x56

```