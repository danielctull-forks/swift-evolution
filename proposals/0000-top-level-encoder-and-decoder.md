# TopLevelEncoder and TopLevelDecoder Protocols

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Daniel Tull](https://github.com/danielctull), [Jasdev Singh](https://github.com/jasdev)
* Review Manager:
* Status:
* Implementation: [WIP branch](https://github.com/apple/swift/compare/master...jasdev:master)

## Introduction

This proposal introduces `TopLevelEncoder` and `TopLevelDecoder` protocols to the Standard Library (currently located in Combine), which are useful for representation-agnostic encoding and decoding of data.

Swift Evolution pitch thread: [Move Combine’s TopLevelEncoder and TopLevelDecoder protocols into the standard library](https://forums.swift.org/t/move-combines-toplevelencoder-and-topleveldecoder-protocols-into-the-standard-library/32494)

## Motivation

The following can apply to both decoding and encoding, but to prevent repetition we will focus on decoding.

A function may want to be able to decode data, but not know the implementation details of the specific encoding used. Consider an open-source networking package with a type to wrap the details of a network request.

```swift
struct Resource<Value> {
    let request: URLRequest
    let transform: (Data, URLResponse) throws -> Value
}
```

It may want to include an initializer for decoding a decodable type using a decoder, but be agnostic to the format of the data, so it also specifies a protocol. The package also defines conformance for the decoders in Foundation, `JSONDecoder` and `PropertyListDecoder`.

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
        // Implementation of YAML decoder.
    }
}
```

For `YAMLDecoder` to conform to `Decoder`, it would have to import the networking package which isn’t great because users of the YAML package may not necessarily wish to do so.

Users of both packages have to conform to a protocol they don’t control and further, possible changes to the library might break existing conformances.

And lastly, the Combine team relayed their intent on these two protocols living in Swift proper:

> [\[The Combine authors\] did intend for `TopLevelEncoder` and `TopLevelDecoder` to be Standard Library types, if possible. I support moving them down, especially now that we have the compiler feature to let us do it.](https://forums.swift.org/t/move-combines-toplevelencoder-and-topleveldecoder-protocols-into-the-standard-library/32494/9)

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

These protocols can be adopted by packages defining new decoders or encoders and those wanting the functionality described above.

## Detailed design

The `JSONDecoder` and `PropertyListDecoder` types in Foundation should be made to conform to the `TopLevelDecoder` protocol.

Likewise, the `JSONEncoder` and `PropertyListEncoder` types in Foundation should be made to conform to the `TopLevelEncoder` protocol.

## Source compatibility

This is a purely additive change. We can similarly lean on the [shadowing work](https://github.com/apple/swift-evolution/blob/b394ae8fff585c8fdc27a50422ea8a90f13138d2/proposals/0235-add-result.md#source-compatibility) from SE-0235 that allowed `Result` to be added to the Standard Library.

## Effect on ABI stability

This is a purely additive change.

## Effect on API resilience

This has no impact on API resilience which is not already captured by other language features.

## Alternatives considered

None.

