# ADR 11: Refactor `ibc-rs`'s Error Handling Architecture

## Changelog

- 2024-04-18: Draft Proposed

## Status

## Context

Errors in `ibc-rs` should accomplish one of two possible purposes:

1. They enable users to implement logic for responding to particular types of errors that occur.
2. They provide a detailed human-readable report of an error stack for debugging purposes.

Any error types that are encoded as enums should be addressing (1). This implies that every error enum variant should
be one that users are actually able to respond to. However, `ibc-rs`'s myriad `Error` types (see [ics07][ics07-error] and [ics25][ics25-error] errors) expose
too many variants that are too specific; most of them are not errors that would ever be exposed to users, much less
reacted to with bespoke logic. Since it's unrealistic to expect that users would handle these errors, they should regarded
as internal protocol errors that aim to accomplish (2).

### Proposal

In light of this rationale, this ADR proposes a restructuring of `ibc-rs`'s error types such
that each adheres to one and only one classification: protocol errors and host errors.

#### Protocol Errors

These errors are defined within `ibc-rs`, and are emitted with the goal of building
up a helpful stack trace when an error occurs.

The top-level type that encapsulates all protocol errors would be a cleaned up version
of the current [`ContextError`][context-error] type. The main differences between
the new `ProtocolError` type and the current `ContextError` type are that:

- it would no longer include error variants for representing host errors
- its purpose is solely to generate clear error messages for debugging

Thus, protocol errors are not ones that we expect users to handle.

#### Host Errors

These errors are defined and controlled by hosts. They should ideally only be returned
from `ValidationContext`/`ExecutionContext` methods, and are defined as associated
types on those contexts:

```diff
use ibc_core::error::ProtcolError as IbcProtoclError;

pub trait ValidationContext {
+    type Error: From <IbcProtcolError>;

-    fn host_timestamp(&self) -> Result<Timestamp, ContextError>;
+    fn host_timestamp(&self) -> Result<Timestamp, Self::Error>;
}
```

```rust
// Example of a host-defined error type
enum HostError {
    Ibc(IbcProtcolError),
    // Other error variants relevant to the host
    ...
}

impl From<IbcProtocolError> for HostError {
    fn from(ibc_error: IbcProtocolError) -> Self {
        Self {
             Ibc(ibc_error)
        }
    }
}
```

Host-defined error types would wrap `ibc-rs`'s `ProtocolError` type and implement
`From<IbcProtocolError>` in order to facilitate conversion between them.

## Decision

[Where should this top-level error type reside?]
[Should ValidationContexts implemented on apps (i.e. the TokenTransfer app) also implement an `Error` associated type?]

## Tradeoffs

### Positive

### Negative

## References

[ics07-error]: https://github.com/cosmos/ibc-rs/blob/4ea4dcb863efa12f5628a05588e2207112035e4a/ibc-clients/ics07-tendermint/types/src/error.rs#L19
[ics25-error]: https://github.com/cosmos/ibc-rs/blob/4ea4dcb863efa12f5628a05588e2207112035e4a/ibc-core/ics25-handler/types/src/events.rs#L16
[context-error]: https://github.com/cosmos/ibc-rs/blob/3a4acfd64d80277808ba0e8cc5ff1c50ca6f7966/crates/ibc/src/core/context.rs#L74
