# TopLevelEncoder and TopLevelDecoder Protcols

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Daniel Tull](https://github.com/danielctull), [Jasdev Singh](https://github.com/jasdev)
* Review Manager: TBD
* Status: **Awaiting implementation**

*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)

## Introduction

This proposal introduces `TopLevelEncoder` and `TopLevelDecoder` protocols to the standard library, which are useful for creating general purpose code around the concepts of encoding and decoding data.

Swift-evolution thread: [Move Combine’s TopLevelEncoder and TopLevelDecoder protocols into the standard library](https://forums.swift.org/t/move-combines-toplevelencoder-and-topleveldecoder-protocols-into-the-standard-library/32494)

## Motivation

The following can apply to both decoding and encoding, but to prevent repetition we will focus on a decoding example.

A function may want to be able to decode data, but not know the implementation details of the specific encoding used. Consider an open source networking package with a type to wrap the details of a network request.

```swift
struct Resource<Value> {
    let request: URLRequest
    let transform: (Data, URLResponse) throws -> Value
}
```

It may want to include an initializer for decoding a decodable type using a decoder, but be agnostic to the specific format of the data, so it also specifies a protocol. The package also defines conformance for the decoders found in Foundation, `JSONDecoder` and `PropertyListDecoder`.

```swift
protocol Decoder {
    func decode<T>(_ type: T.Type, from: Data) throws -> T where T: Decodable
}

extension JSONDecoder: Decoder {}
extension PropertyListDecoder: Decoder {}

extension Resource where Value: Decodable {
    init<D: Decoder>(request: URLRequest, value: Value.Type, decoder: D) {
        self.init(request: request) { data, _ in
            try decoder.decode(Value.self, from: data)
        }
    }
}
```

This is fine if the caller wishes to use it for JSON or property list formatted data. However, they may have data defined as YAML and thus choose to use a package that provides a `YAMLDecoder`.

```swift
class YAMLDecoder {
    init() {}
    func decode<T>(_ type: T.Type, from: Data) throws -> T where T: Decodable {
        // Implementation of yaml decoder.
    }
}
```

For `YAMLDecoder` to conform to the networking library's `Decoder` protocol, it would have to import the networking package which is undesirable because users of the YAML package may not necessarily wish to use that particular networking package.

Therefore the user of both pacakges is made to add the conformance for a type they do not own to a protocol they do not control, which brings issues about changes to either library that might break such a conformance.

## Proposed solution

Introduce new `TopLevelDecoder` and `TopLevelEncoder` protocols:

```swift
/// A type that defines methods for decoding.
public protocol TopLevelDecoder {

    /// The type this decoder accepts.
    associatedtype Input

    /// Decodes an instance of the indicated type.
    func decode<T>(_ type: T.Type, from: Self.Input) throws -> T where T: Decodable
}

/// A type that defines methods for encoding.
public protocol TopLevelEncoder {

    /// The type this encoder produces.
    associatedtype Output

    /// Encodes an instance of the indicated type.
    func encode<T>(_ value: T) throws -> Self.Output where T : Encodable
}
```

These protocols can be adopted by packages defining new decoders/encoders and used by packages wanting their functionality as described above. A user of two independent pacakges will then just be able to use them together with no overhead because of these common protocols defined in the standard library.

## Detailed design

The `JSONDecoder` and `PropertyListDecoder` types in Foundation should be made to conform to the `TopLevelDecoder` protocol.

Likewise, the `JSONEncoder` and `PropertyListEncoder` types in Foundation should be made to conform to the `TopLevelEncoder` protocol.

## Source compatibility

This is a purely additive change.

## Effect on ABI stability

This is a purely additive change.

## Effect on API resilience

This has no impact on API resilience which is not already captured by other
language features.

## Alternatives considered

No others.

