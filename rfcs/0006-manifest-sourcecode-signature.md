# RFC-0006: Manifest Metadata

<dl>
  <dt>Author</dt>
  <dd>David Mihal</dd>

  <dt>RFC pull request</dt>
  <dd><a href="https://github.com/graphprotocol/rfcs/pull/27">https://github.com/graphprotocol/rfcs/pull/27</a></dd>

  <dt>Date of submission</dt>
  <dd>2022-06-29</dd>
</dl>

## Summary

Add optional metadata to the subgraph manifest, containing the source-code for
mapping files and/or a cryptographic signature from the developer.

## Goals & Motivation

CryptoStats is currently building tooling to improve the developer experience
for building subgraphs. As part of this initiative, we are aiming to make the
development of subgraphs more “collaborative”, allowing subgraph authors to
be more easily identified, and providing an easy method for subgraphs to be
“forked”. Adding these features necessitates adding additional metadata to the
traditional YAML manifest file. This document aims to standardize these
attributes, allowing other tooling in the Graph ecosystem to take advantage of
them.

## Urgency

While this feature is low-priority, CryptoStats _will_ be launching a tool that
uses these features soon.

## Terminology

- **Manifest** - the YAML file describing all attributes of a subgraph.
- **Entry point** - the root file to be used when compiling source code.

## Detailed Design

We propose adding two optional sections to the Manifest file: `sourceCode` and
`signature`.

### Source Code

The manifest provides most files needed for a developer to re-build their own
subgraph, such as the schema document, all smart contracts and ABIs. However,
the mapping file is only provided as compiled WASM bytecode, which can not be
modified by other developers.

We would like to propose the addition of an optional sourceCode section to the
manifest, containing a list of entities with the following attributes: 

| Field | Type | Description |
| --- | --- | --- |
| **file** | [*Path*](#16-path) | The path of the compiled output file (typically an [IPLD link](https://github.com/ipld/specs/) matching the link used elsewhere in the file) |
| **source** | [*Path*](#16-path) | The path of the source code file or directory |
| **name** | optional *String* | Optional "file name" for the source file (if single file) |
| **entryPoint** | optional *String* | If the `source` path is a directory, `entryPoint` should be provided to define the entry point file for compilation |

#### Example

```yml
specVersion: 0.0.2
repository: 'https://github.com/graphprotocol/uniswap-v2-subgraph'
dataSources:
  - kind: ethereum/contract
    mapping:
      abis:
        - file:
            /: /ipfs/QmZ55G1yYFzde8Vcq4cpLfNgPSEibpLi9aYCqS1jEvCKQ9
          name: Factory
        - file:
            /: /ipfs/QmXuTbDkNrN27VydxbS2huvKRk62PMgUTdPDWkxcr2w7j2
          name: ERC20
      apiVersion: 0.0.3
      entities:
        - Pair
        - Token
      eventHandlers:
        - event: 'PairCreated(indexed address,indexed address,address,uint256)'
          handler: handleNewPair
      file:
        /: /ipfs/QmZLfiUaV9CR62UQRccEoJM6fJa5p219rvXiwi3FQVpnny
      kind: ethereum/events
      language: wasm/assemblyscript
    name: Factory
    network: mainnet
    source:
      abi: Factory
      address: '0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f'
      startBlock: 10000834
schema:
  file:
    /: /ipfs/QmRkCTHhWGGSb1JnAscsWNtCeEkymLrrF2ufX9yEVTXW6H
sourceCode:
  # If a mapping is compiled from a single file, the name attribute may be included
  - fileName: mapping.ts
    file: /ipfs/QmZLfiUaV9CR62UQRccEoJM6fJa5p219rvXiwi3FQVpnny
    source: /ipfs/QmM6fJa5p219rvXiwi3FQVpnnyZLfiUaV9CR62UQRccEoJ
  # If a mapping is compiled from multiple files, source can be a directory
  - file: /ipfs/QmZLfiUaV9CR62UQRccEoJM6fJa5p219rvXiwi3FQVpnny
    source: /ipfs/QmM6fJa5p219rvXiwi3FQVpnnyZLfiUaV9CR62UQRccEoJ
    # If a directory is provided, an entryPoint attribute must be specified
    entryPoint: pool.ts
```

### Signature

As subgraphs become a more important piece of the blockchain stack, there may
be value in having a cryptographic signature attached to a subgraph. This
allows any user to verify that a subgraph was created by a specific account.

We propose adding an optional `signature` section to the manifest with the following attributes: 

| Field | Type | Description |
| --- | --- | --- |
| **signer** | *String* | The Ethereum address that signed the manifest  |
| **signature** | *String* | The hex-encoded ECDSA signature string of the prefixed hash of the manifest (see below) |

#### Example

```yml
specVersion: 0.0.2
description: Uniswap is a decentralized protocol for automated token exchange on Ethereum.
repository: 'https://github.com/graphprotocol/uniswap-v2-subgraph'
dataSources:
  ...
signature:
  signer: 0x8f73be66ca8c79382f72139be03746343bf5faa0
  signature: 0xb694cf5b76776702b8430298665e02eef435b5616a57f4ac309fa48a4a9943ed26bad253b520cc5de9c298ec90da87a59bed1a63f545353d83e88491c8ed78cf1c
```

#### Signature generation

The signature is generated by taking the keccak256 hash of the manifest string,
prefixing it with the string `Subgraph hash: `, then taking a ECDSA signature
of this string.

The resulting signature is then included in the new “signature” section and
appended to the manifest.

The following code is an example of how a signature can be generated using
ethers.js:

```js
const manifest = `specVersion: 0.0.2
dataSources:
...`;

const hash = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(manifestCode));
const message = `Subgraph hash: ${hash}`;
const signature = await signer.signMessage(message);

const newManifestCode = `${manifestCode}
signature:
  signer: ${account}
  signature: ${signature}
`
```

#### Signature validation

To validate the signature attached to a subgraph, the signing process must be
reversed, removing the signature section before validating.

The following code is an example of how to validate a manifest signature using
ethers.js:

```js
const SIGNATURE_REGEX = /\nsignature:\n  signer: ['"]?(0x[0-9a-fA-F]{40})['"]?\nsignature:['"]?(0x[0-9a-fA-F]{130})['"]?\n/

const signatureResults = SIGNATURE_REGEX.exec(code);
if (!signatureResults) {
  throw new Error('Signature not found');
}

const [signatureSection, signer, signature] = signatureResults;

const signedManifest = code.replace(signatureSection, ''); // Remove signature from manifest
const hash = ethers.utils.keccak256(ethers.utils.toUtf8Bytes(signedCode));
const message = `Subgraph hash: ${hash}`;
const verifiedSigner = ethers.utils.verifyMessage(message, signature);

if (verifiedSigner.toLowerCase() !== signer.toLowerCase()) {
  throw new Error('Invalid signature');
}
console.log(`Subgraph verified signed by ${signer}`)
```

## Compatibility

The proposed design is fully backwards compatable.

## Drawbacks and Risks

This proposal only provides additional metadata for subgraphs, therefore there
are no risks nown at this time.

## Alternatives

Currently, subgraphs may include a link to a GitHub repo containing source
code. However, this does not allow for programatic managment & compilation
of source code. Furthermore, many GitHub repos may fall out of date compared
to the published subgraphs.

## Open Questions

- None.
