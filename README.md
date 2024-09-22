# evm-signatures

## signatures

Before eip-191 and eip-712 standards we used to get the signer like this:

```
function getSignerSimple(uint256 message, uint8 _v, bytes32 _r, bytes32 _s) public pure returns (address) {
  bytes32 hashedMessage = bytes32(message); // if message was a string, we would use keccak256(abi.encodePacked(message))
  address signer = ecrecover(hashedMessage, _v, _r, _s);
  return signer;
}
```

### eip-191

```
0x19 <1 byte version> <version specific data> <data to sign>
```
```
0x19 is the prefix, which means that the data is a signature. And due to how Ethereum transactions are encoded, it ensures that the data associated with the signed message cannot be a valid Ethereum transaction.
```

```
<1 bytes version> 
Version that the signed data is using. This allows different version to different signed data structures
The allowed values are:

0x00: Data with intended validator (address validating the signature is validated here)
0x01: Structured Data (associated with EIP-712)
0x45: personal_sign messages
```

```
<version specific data>
data associated with the version
For example for 0x01, this is where we specify the validator address
```

```
<data to sign>
Message to sign
```

The getSigner code would like this after eip-191

```
function getSigner191(uint256 message, uint8 _v, bytes32 _r, bytes32 _s) public view returns (address) {

  // Arguments when calculating hash to validate
  // 1: byte(0x19) - the initial 0x19 byte
  // 2: byte(0) - the version byte
  // 3: version specific data, for version 0, it's the intended validator address
  // 4-6: application specific data

  bytes1 prefix = bytes1(0x19);
  bytes1 eip191Version = bytes1(0);
  address intendedValidatorAddress = address(this);
  bytes32 applicationSpecificData = bytes32(message); // if message was a string, we would use keccak256(abi.encodePacked(message))

  // 0x19 <1 byte version> <version specific data> <data to sign>
  bytes32 hashedMessage = keccak256(abi.encodePacked(prefix, eip191Version, intendedValidatorAddress, applicationSpecificData));

  address signer = ecrecover(hashedMessage, _v, _r, _s);
  return signer;
```

### eip-712

```
0x19 0x01 <domainSeperator> <hashStruct(message)>
```

```
<domainSeperator>
This is the version specific data. It represents the hash of the struct defining the domain of the message being signed
<domainSeperator> = <hashStruct(eip712Domain)>

struct eip712Domain {
  string name;
  string version;
  uint256 chainId;
  address verifyingContract;
  bytes32 salt;
}

It can be written as:
0x19 0x01 <hashStruct(eip712Domain)> <hashStruct(message)>
```

```
hashStruct is the hash of the type hash + the hash of the type itself

// Hash of our EIP712 domain struct
bytes32 constant EIP712DOMAIN_TYPEHASH = keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");

// Define what our "domain" struct looks like
eip_712_domain_seperator_struct = EIP712Domain({
  name: "SignatureVerifier",
  version: "1",
  chainId: 1,
  verifyingContract: address(this)
});

domain_seperator = keccak256(
  abi.encode(
    EIP712DOMAIN_TYPEHASH,
    keccak256(bytes(eip_712_domain_seperator_struct.name)),
    keccak256(bytes(eip_712_domain_seperator_struct.version)),
    eip_712_domain_seperator_struct.chainId,
    eip_712_domain_seperator_struct.verifyingContract
  )
);
```

```
<hashStruct(message)>

struct Message {
  uint256 number;
}

bytes32 public constant MESSAGE_TYPEHASH = keccak256("Message(uint256 number)");

bytes hashedMessage = keccak256(
  abi.encode(
    MESSAGE_TYPEHASH,
    Message({number: message})
    )
);
```

The getSigner code would like this after eip-712:

```
function getSigner712(uint256 message, uint8 _v, bytes32 _r, bytes32 _s) public view returns (address) {

  // Arguments when calculating hash to validate
  // 1: byte(0x19) - the initial 0x19 byte
  // 2: byte(1) - the version byte
  // 3: hashStruct of domain seperator (includes the typehash of the domain struct)
  // 4: hashstruct of message (includes the typehash of the message struct)

  bytes1 prefix = bytes1(0x19);
  bytes1 eip712Version = bytes1(1);
  bytes32 hashStructOfDomainSeperator = domain_seperator;

  bytes32 hashedMessage = keccak256(
    abi.encode(
      MESSAGE_TYPEHASH,
      Message({number: message})
      )
  );

  bytes32 digest(keccak256(abi.encodePacked(prefix, eip19Version, hashStructOfDomainSeperator, hashedMessage);

  address signer = ecrecover(digest, _v, _r, _s);
  return signer;
}
```

Using OpenZeppelin to do it would look like this:

```
function getSigner712UsingOZ(uint256 _message, uint8 _v, bytes32 _r, bytes32 _s) public view returns (address) {

  bytes32 hashedMessage = _hashTypedDataV4(
                            keccak256(
                              abi.encode(
                                MESSAGE_TYPEHASH,
                                Message({message: _message})
                              )
                            )
                          );

(address signer, /*ECDSA.RecoverError recoverError*/, /*bytes32 signatureLength*/ = 
  ECDSA.tryRecover(hashedMessage, _v, _r, _s);

return signer;
}
```



