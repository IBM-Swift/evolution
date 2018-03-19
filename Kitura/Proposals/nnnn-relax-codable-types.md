## Relax Type Constraints for Codable Routing
* Proposal: KIT-nnnn
* Authors: [David Jones](https://github.com/djones6)
* Review Manager: [Lloyd Roseblade](https://github.com/lroseblade)
* Status: DRAFT
* Decision Notes: Rationale, Additional Commentary
* Previous Revision: 1
* Previous Proposal: N/A

### Introduction
Codable Routing currently requires that types used as Input or Output conform to `Codable`, ie. `Encodable & Decodable`. This proposal is to relax the requirements on those types to simply `Decodable` for Input and `Encodable` for Output types.

### Motivation
Users are currently required to support both Encodable and Decodable when creating types for use with Codable Routing. Whilst the compiler can often synthesise the encode/decode functions, there are cases where the user must provide an implementation, such as an enum with associated values.

If the user wants to use that type only in one direction - say if they want to receive data in a type-safe way from an existing service - they are obliged to implement an unnecessary encode/decode function just to satisfy the type requirements.

Example: https://github.com/IBM-Swift/Kitura/issues/1235

### Proposed solution
In the Codable Router extension and its associated closure aliases,
- Replace each instance of `<I: Codable>` with `<I: Decodable>`
- Replace each instance of `<O: Codable>` with `<O: Encodable>`

This is not an API-breaking change:
- `Decodable` and `Encodable` are a subset of `Codable`, so existing Codable types can be used, and can continue to be used for both the input and output type
- Kitura's `Router` class is not `open` (and so cannot have been subclassed outside of the Kitura package).

### Detailed design
A working prototype exists here:
- https://github.com/IBM-Swift/Kitura/compare/issue_1235
- https://github.com/IBM-Swift/KituraContracts/compare/issue_1235

### Alternatives considered
None
