Change Log
=======

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](http://keepachangelog.com/)
and this project adheres to [Semantic Versioning](http://semver.org/).

# [unreleased]

## Added

- `cfdppy.handler.source.acknowledge_inactive_finished_pdu`, the counterpart of the existing
  `acknowledge_inactive_eof_pdu`. A source handler resets as soon as it has queued the ACK for
  the Finished PDU, so it retains no state to answer a retransmission of that Finished PDU if
  the ACK was lost. Without an acknowledgment the receiving entity retransmits to its positive
  ACK limit and declares `POSITIVE_ACK_LIMIT_REACHED` at the end of a transfer which actually
  succeeded.

## Fixed

- The destination handler no longer livelocks when the ACK for its Finished PDU never arrives.
  Reaching the positive ACK limit cancelled the transaction, which re-ran the notice of
  completion, re-sent the Finished PDU and restarted the positive ACK procedure with a fresh
  counter, so the handler never left `WAITING_FOR_FINISHED_ACK`: it emitted a Finished PDU and a
  transaction finished indication on every single expiration, for as long as the process lived,
  and could never accept another transaction. Per CFDP 4.11.2.2.3 a fault declared while
  transferring the cancel PDU now abandons the transaction, which is the rule the source handler
  already applied to its own EOF (cancel) PDU. A fault handler configured as
  `ABANDON_TRANSACTION` for `POSITIVE_ACK_LIMIT_REACHED` also no longer crashes the positive ACK
  procedure with an `AttributeError`.
- A retransmitted file data PDU which exactly refilled the most recently received window no
  longer leaves its gap in the lost segment tracker. The removal was decided by comparing the end
  of the received segment against the *start* of that window rather than its end, a condition such
  a retransmission never satisfies, so the destination re-requested data it had already written on
  every NAK round until it reached its NAK limit, with the transfer stuck at full progress.
- `_AckedModeParams.lost_seg_tracker` used a mutable dataclass default, so a single
  `LostSegmentTracker` was created at class definition time and shared by every parameter set:
  all destination handlers in a process, and every transaction of a single handler, accumulated
  their lost segments in the same list.
- The destination handler now re-acknowledges a duplicate EOF PDU received in the
  `WAITING_FOR_MISSING_DATA`, `TRANSFER_COMPLETION`, `SENDING_FINISHED_PDU` and
  `WAITING_FOR_FINISHED_ACK` steps. Per CFDP 4.7.2 every EOF PDU must be acknowledged, and a
  duplicate means the previous ACK was lost: the sender retransmits the EOF on its positive ACK
  timer and would otherwise reach its limit while the receiver silently discarded every copy.
  The transaction state is left untouched, only the ACK is re-issued, and the ACK carries the
  condition code of the EOF PDU it acknowledges.
- A metadata PDU which arrives while the deferred lost segment procedure is active no longer
  moves the destination handler back to `RECEIVING_FILE_DATA`. The EOF PDU has already been
  handled and acknowledged at that point, so the sender has no reason to send another one and
  the handler waited for it forever with its remaining gaps never re-requested. It now continues
  in `WAITING_FOR_MISSING_DATA`.
- The destination handler retains the checksum of an EOF PDU which arrived before the metadata
  PDU. It was dropped, so a transaction which recovered from a lost metadata PDU completed
  against the empty default checksum and reported `FILE_CHECKSUM_FAILURE` for a file that had
  arrived intact.

# [v0.7.0] 2026-09-08

- Bump allowed `spacepackets` to >=0.30, <0.33
- Added Python 3.14 to CI

## Removed

- Dropped Python 3.9 support. It reached end-of-life on 2025-10-31. Minimum is now Python 3.10.

## Fixed

- Metadata-only transactions (e.g. a Proxy Put Request per CCSDS 727.0-B-5 6.1) now correctly
  send and expect an EOF (No error) PDU, as required by 4.6.1.1.9 case (C). The source handler no
  longer tries to checksum a source file that does not exist for this case, and the destination
  handler no longer jumps straight to transfer completion without waiting for the EOF PDU.
- Source handler's `_prepare_file_params` did not set `file_size` for a metadata-only put
  request. This was masked for a freshly constructed handler, but a handler reused for a second,
  metadata-only transaction after a first one completed raised an `AssertionError`.

# [v0.6.0] 2025-09-25

- Bump allowed `spacepackets` to >=0.30, <=0.31
- Transition CRC library from `crcmod` to `fastcrc`

## Removed

- `TransactionStep.SENDING_ACK_OF_FINISHED` for source handler which is not required anymore.

## Changed

- Replaced `*Cfg*` abbreviation with `*Config*`

## Fixed

- Corrections for EOF (Cancel) Handling: Perform proper checks on whether the file is actually
  completed using the supplied file size and checksum.
- Some fixes for the handling of multiple NAK PDUs in one NAK sequence

# [v0.5.1] 2025-02-10

- Bump allowed `spacepackets` to v0.28.0

# [v0.5.0] 2025-01-17

## Added

- Added `RestrictedFilestore` to limit the file access of the `NativeFilestore` to a specific
  directory.
- `Filestore` creates a directory if it does not exist when creating a new file.

## Fixed

- Correction for `InvalidDestinationId` exception arguments in destination handler.
- Destination handler now only checks entity ID values when checking inserted packets.
- Source handler used an incorrect check if the file exists without the virtual filestore.
- Source handler opened files without the virtual filestore

# [v0.4.0] 2024-11-08

## Added

- `progress` property for both source and destination handler to track the progress of a
  transaction.
- `file_size` property for both destination and source handler.
- `get_pdu_request` getter function to retrieve the active Put Request for the source handler.

# [v0.3.0] 2024-10-15

## Changed

- Simplified state machine usage: Packets are now inserted using an optional `packet` argument
  of the `state_machine` call.
- Removed some of the visible intermedia transaction steps. For example, instead of remaining
  on `TransactionStep.SENDING_FINISHED_PDU`, the destination handler will still generate the
  Finished PDU but jump to the next step immediately without requiring another state machine call.
  For the source handler, the same was done for the `TransactionStep.SENDING_EOF` step.

## Removed

- `insert_packet` API of the source and destination handler. Packet insertion is now performed
  using the `state_machine` call.

## Fixed

- Fault location field of the Finished PDU is now set correctly for transfer cancellations.

# [v0.2.0] 2024-08-27

## Fixed

- The large file flag was not set properly in the source handler for large file transfers.
- The CRC algorithms will now be used for empty files as well instead of hardcoding the
  checksum type to the NULL checksum. This was a bug which did not show directly for
  checksums like CRC32 because those have an initial value of 0x0

## Changed

- Added `file_size` abstract method to `VirtualFilestore`
- Renamed `HostFilestore` to `NativeFilestore`, but keep old name alias for backwards compatibility.
- Added `calculate_checksum` and `verify_checksum` to `VirtualFilestore` interface.

# [v0.1.2] 2024-06-04

Updated documentation configuration to include a `spacepackets` docs mapping. This
should fix references to the `spacepackets` documentation.

# [v0.1.1] 2024-04-23

- Allow `spacepackets` range from v0.23 to < v0.25

# [v0.1.0]

Initial release of the `cfdp-py` library which was split off the
[tmtccmd library](https://github.com/robamu-org/tmtccmd).

[unreleased]: https://github.com/us-irs/cfdp-py/compare/v0.7.0...HEAD
[v0.7.0]: https://github.com/us-irs/cfdp-py/compare/v0.6.0...v0.7.0
[v0.6.0]: https://github.com/us-irs/cfdp-py/compare/v0.5.1...v0.6.0
[v0.5.1]: https://github.com/us-irs/cfdp-py/compare/v0.5.0...v0.5.1
