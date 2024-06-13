# ADR 11: Refactor `ibc-rs`'s Error Handling Architecture

## Changelog

- 2024-04-18: Draft Proposed

## Status

Proposed

## Context

Errors in `ibc-rs` should accomplish one of two possible purposes:

1. They enable users to implement logic for responding to particular types of errors that occur.
2. They provide a detailed human-readable report of an error stack for debugging purposes.

Any error types that are encoded as enums should be addressing (1). This implies that every error enum variant should
be one that users are actually able to respond to. However, `ibc-rs`'s myriad `Error` types (see [ics07][ics07-error] and [ics25][ics25-error] errors for examples) expose
too many variants that are too specific; most of them are not errors that would ever be exposed to users, much less
reacted to with bespoke logic. Since it's unrealistic to expect that users would handle these errors, they should be regarded
as internal protocol errors that aim to accomplish (2).

### Proposal

In light of this rationale, this ADR proposes a restructuring of `ibc-rs`'s error types such
that each error type adheres to one and only one classification: protocol errors and host errors.

#### Protocol Errors

These errors are defined within `ibc-rs`, and are emitted with the goal of building
up a helpful stack trace when an error occurs.

The top-level type that encapsulates all protocol errors would be a cleaned up version
of the current [`ContextError`][context-error] type; this cleaned up version will be
renamed to `ProtocolError` in order to better capture its IBC-internal nature.
The main differences between the `ProtocolError` type and the current `ContextError` type are that:

- it would no longer include error variants for representing errors that arise from hosts' code
- its purpose is solely to generate clear error messages for debugging

Thus, protocol errors are not ones that we expect users to handle. They should instead provide
nicely-formatted backtrace and source information to aid in the debugging process as much as possible.

#### Host Errors

These errors are defined and controlled by hosts. They should ideally only be returned
from host implementations of `ValidationContext`/`ExecutionContext` methods. We introduce a
`HostError` associated type on those contexts:

```diff
use core::fmt::{Debug, Display};
use std::error::Error as StdError;
use ibc_core_handler_types::error::ContextError;

pub trait ValidationContext {
+    type HostError: Debug + Display + StdError;

-    fn host_timestamp(&self) -> Result<Timestamp, ContextError>;
+    fn host_timestamp(&self) -> Result<Timestamp, ContextError<Self::HostError>>;
}
```

Note that the old `ContextError` type is being renamed to `ProtocolError`. The name
`ContextError` will now be used as `ibc-rs`'s new top-level error type and is defined as such:

```rust
#[derive(Debug, Display)]
pub enum ContextError<E: Debug + Display> {
    /// Host-defined errors.
    Host(E),
    /// Internal protocol-level errors.
    Protocol(ProtocolError),
}
```

This new `ContextError` type captures the `HostError` via a generic parameter, enabling
`ibc-rs`'s logic to react or respond to host-originating errors. This is as opposed to
the current status quo of how host-originating errors are handled, which is that they
are clumsily mapped onto `ibc-rs`'s internal error types. This results in these host-
originating errors not being handled appropriately, as well as contributing to the bloat
of `ibc-rs`'s error enums.

## Outstanding Questions

- What trait bounds should exist on the `HostError` associated type?
  - Just `Display` and `Debug`, or perhaps also `std::error::Error`?
- Should `ValidationContext`s implemented on apps (i.e. the TokenTransfer app) also implement a `HostError` associated type?

## Decision

## Tradeoffs

### Positive

### Negative

## References

[ics07-error]: https://github.com/cosmos/ibc-rs/blob/4ea4dcb863efa12f5628a05588e2207112035e4a/ibc-clients/ics07-tendermint/types/src/error.rs#L19
[ics25-error]: https://github.com/cosmos/ibc-rs/blob/4ea4dcb863efa12f5628a05588e2207112035e4a/ibc-core/ics25-handler/types/src/events.rs#L16
[context-error]: https://github.com/cosmos/ibc-rs/blob/3a4acfd64d80277808ba0e8cc5ff1c50ca6f7966/crates/ibc/src/core/context.rs#L74
