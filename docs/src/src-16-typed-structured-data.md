# SRC-16: Typed Structured Data

The following standard sets out to standardize encoding and hashing of typed structured data. This enables secure off-chain message signing with human-readable data structures.


## Motivation

As the Fuel ecosystem expands, there's an increasing need for applications to handle complex, human-readable data structures rather than raw bytes. When users sign messages or transactions, they should be able to clearly understand what they're signing, whether it's a simple asset transfer, or a complex DeFi interaction. Without a standard method for hashing structured data, developers risk implementing their own solutions, which could lead to confusion or compromise security. This standard provides a secure and consistent way to handle encoding and hashing of structured data, ensuring both safety and usability within ecosystem.


This standard aims to:

* Provide a secure, standardized method for hashing structured data
* Enable clear presentation of structured data for user verification during signing
* Support complex data types that mirror Sway structs
* Enable domain separation to prevent cross-protocol replay attacks
* Define a consistent encoding scheme for structured data types
* Remain stateless, not requiring any storage attributes to enable use across all Fuel program types.


## Prior Art

This standard uses ideas from [Ethereum's EIP-712 standard](https://eips.ethereum.org/EIPS/eip-712), adapting its concepts for the Fuel ecosystem. EIP-712 has proven successful in enabling secure structured data signing for applications like the various browser based wallets and signers that are utilized throughout various DeFi protocols.

## Specification

### Definition of Typed Structured Data 𝕊:

The set of structured data 𝕊 consists of all instances of struct types that can be composed from the following types:

Atomic Types:
```sway
u8 to u256
bool
b256 (hash)
```

Dynamic Types:
```sway
Bytes   // Variable-length byte sequences
String  // Variable-length strings
```

Reference Types:

Arrays (both fixed size and dynamic)
Structs (reference to other struct types)


Example struct definition:

```sway
struct Mail {
    from: Address,
    to: Address,
    contents: String,
}
```

### Domain Separator Encoding

The domain separator provides context for the signing operation, preventing cross-protocol replay attacks. It is computed as hashStruct(domain) where domain is defined as:

```sway
pub struct SRC16Domain {
    name: String,                   // The protocol name (e.g., "MyProtocol")
    version: String,                // The protocol version (e.g., "1")
    chain_id: u64,                  // The Fuel chain ID
    verifying_contract: ContractId, // The contract id that will verify the signature
}
```

The encoding follows this scheme:

* Add SRC16_DOMAIN_TYPE_HASH
* Add Keccak256 hash of name string
* Add Keccak256 hash of version string
* Add chain ID as 32-byte big-endian
* Add verifying contract id as 32 bytes


## Type Encoding

Each struct type is encoded as name ‖ "(" ‖ member₁ ‖ "," ‖ member₂ ‖ "," ‖ … ‖ memberₙ ")" where each member is written as type ‖ " " ‖ name.

Example:

```
Mail(address from,address to,string contents)
```

## Data Encoding

### Definition of hashStruct

The hashStruct function is defined as:

hashStruct(s : 𝕊) = keccak256(typeHash ‖ encodeData(s))
where:

* typeHash = keccak256(encodeType(typeOf(s)))
* ‖ represents byte concatenation
* encodeType and encodeData are defined below


### Definition of encodeData

The encoding of a struct instance is enc(value₁) ‖ enc(value₂) ‖ … ‖ enc(valueₙ), the concatenation of the encoded member values in the order they appear in the type. Each encoded member value is exactly 32 bytes long.

The values are encoded as follows:

Atomic Values:
* Boolean false and true are encoded as u64 values 0 and 1, padded to 32 bytes
* Addresses, ContractId, Identity, and b256 are encoded directly as 32 bytes
* Unsigned Integer values (u8 to u256) are encoded as big-endian bytes, padded to 32 bytes

Dynamic Types:
* Bytes and String are encoded as their Keccak256 hash

Reference Types:
* Arrays (both fixed and dynamic) are encoded as the Keccak256 hash of their concatenated encodings
* Struct values are encoded recursively as hashStruct(value)

The implementation of `TypedDataHash` for `𝕊` SHALL utilize the `DataEncoder` for encoding each element of the struct based on its type.

## Final Message Encoding

The encoding of structured data follows this pattern:

encode(domainSeparator : 𝔹²⁵⁶, message : 𝕊) = "\x19\x01" ‖ domainSeparator ‖ hashStruct(message)

where:

* \x19\x01 is a constant prefix
* ‖ represents byte concatenation
* domainSeparator is the 32-byte hash of the domain parameters
* hashStruct(message) is the 32-byte hash of the structured data



## Example implementation:

```sway
const MAIL_TYPE_HASH: b256 = 0x536e54c54e6699204b424f41f6dea846ee38ac369afec3e7c141d2c92c65e67f;

impl TypedDataHash for Mail {

    fn type_hash() -> b256 {
        MAIL_TYPE_HASH
    }

    fn struct_hash(self) -> b256 {
        let mut encoded = Bytes::new();
        encoded.append(
            MAIL_TYPE_HASH.to_be_bytes()
        );
        encoded.append(
            DataEncoder::encode_address(self.from).to_be_bytes()
        );
        encoded.append(
            DataEncoder::encode_address(self.to).to_be_bytes()
        );
        encoded.append(
            DataEncoder::encode_string(self.contents).to_be_bytes()
        );

        keccak256(encoded)
    }
}
```


## Rationale

* Domain separators provides protocol-specific context to prevent signature replay across different protocols and chains.
* Type hashes ensure type safety and prevent collisions between different data structures
* The encoding scheme is designed to be deterministic and injective
* The standard maintains compatibility with existing Sway types and practices


## Backwards Compatibility

This standard is compatible with existing Sway data structures and can be implemented alongside other Fuel standards. It does not conflict with existing signature verification methods.

### Type System Compatibility Notes

When implementing SRC16 in relation to EIP712, the following type mappings and considerations apply:

#### String Encoding
- Both standards use the same String type and encoding
- SRC16 specifically uses String type only (not Sway's `str` or `str[]`)
- String values are encoded identically in both standards using keccak256 hash

#### Fixed Bytes
- EIP712's `bytes32` maps directly to Sway's `b256`
- Encoded using `encode_b256` in the DataEncoder
- Both standards handle 32-byte values identically
- Smaller fixed byte arrays (bytes1 to bytes31) are not supported in SRC16

#### Address Types
- EIP712 uses 20-byte Ethereum addresses
- When encoding an EIP712 address, SRC16:
  - Takes only rightmost 20 bytes from a 32-byte Fuel Address
  - Pads with zeros on the left for EIP712 compatibility
  - Example: Fuel Address of 32 bytes becomes rightmost 20 bytes in EIP712 encoding

#### ContractId Handling
- ContractId is unique to Fuel/SRC16 (no equivalent in EIP712)
- When encoding for EIP712 compatibility:
  - Uses rightmost 20 bytes of ContractId
  - Particularly important in domain separators where EIP712 expects a 20-byte address

#### Domain Separator Compatibility
```rust
// SRC16 Domain (Fuel native)
pub struct SRC16Domain {
    name: String,                   // Same as EIP712
    version: String,                // Same as EIP712
    chain_id: u64,                  // Fuel chain ID
    verifying_contract: ContractId, // Full 32-byte ContractId
}

// EIP712 Domain (Ethereum compatible)
pub struct EIP712Domain {
    name: String,
    version: String,
    chain_id: u256,
    verifying_contract: b256,      // Only rightmost 20 bytes used
}

Note: When implementing EIP712 compatibility within SRC16, the verifying_contract address in the EIP712Domain must be constructed by taking only the rightmost 20 bytes from either a Fuel ContractId or Address. This ensures proper compatibility with Ethereum's 20-byte addressing scheme in the domain separator.

## Security Considerations

### Replay Attacks:

Implementers must ensure signatures cannot be replayed across:

Different chains (prevented by chain_id)
Different protocols (prevented by domain separator)
Different contracts (prevented by verifying_contract)

### Type Safety:

Implementations must validate all type information and enforce strict encoding rules to prevent type confusion attacks.


## Example Implementation

Example of the SRC-16 implementation where a contract utilizes the encoding scheme to produce a typed structured data hash of the Mail type.

```sway
{{#include ../examples/src16-typed-data/fuel_example/src/main.sw}}

{{#include ../examples/src16-typed-data/ethereum_example/src/main.sw}}
```
