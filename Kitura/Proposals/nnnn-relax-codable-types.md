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

To facilitate receiving a `Decodable` type via a PUT or POST, without requiring an `Encodable` type be returned, this proposal relaxes the rules for responding with an `(Encodable?, RequestError?)`, so that rather than requiring exactly one of these to be non-nil, the following are also permitted:
- `(nil, successStatus)`: respond successfully with the specified status code and no body
- `(nil, nil)`: respond successfully with the default status for that method and no body

Where `successStatus` is a `RequestError` in the 2xx (success) range.

And similarly for `(Identifier?, Encodable?, RequestError?)`:
- `(id, nil, successStatus)`: respond successfully with the Location header, specified status code and no body
- `(id, nil, nil)`: respond successfully with the Location header, default status for that method and no body

where `id` is a valid `Identifier`.

### Detailed design
A working prototype exists here:
- https://github.com/IBM-Swift/Kitura/compare/issue_1235 (https://github.com/IBM-Swift/Kitura/pull/1242)
- https://github.com/IBM-Swift/KituraContracts/compare/issue_1235 (https://github.com/IBM-Swift/KituraContracts/pull/18)

### Alternatives considered
None
