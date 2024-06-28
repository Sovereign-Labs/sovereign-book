# Sequencer Registration via Forced Inclusion

Forced inclusion is a strategic mechanism in rollups designed to circumvent
sequencers that censor user transactions. It allows users to directly submit
transaction batches to the [Data Availability Layer](./da-layer.md) instead of
going through a sequencer.

The Sovereign SDK supports this feature under specific conditions and
guidelines. Crucially, only "Register Sequencer" transactions are accepted for
forced inclusion; all other types will be ignored. For more details, see the
[Rules](#rules) section.

## Usage

The Sovereign SDK limits the number of batches from unregistered sequencers
processed per rollup slot. This measure helps users forced to register as
sequencers due to experiencing censorship and limits the use of this mechanism
as a denial-of-service (DOS) attack vector.

### Process for Forced Registration

1. Create a batch containing a valid "Register Sequencer" transaction.
2. Submit the batch to the Data Availability layer.
3. Nodes collect and execute the transaction.
4. If the transaction complies with all rules, the user is registered as a
   sequencer and can submit regular transaction batches.

## Rules

To ensure forced inclusion requests are processed correctly, the following rules
apply:

- **Transaction Limit**: Only one transaction is allowed per batch. Any
  additional transactions will be discarded.
- **Transaction Type**: The transaction must be a "Register Sequencer"
  transaction.
- **Transaction Construction**: The transaction must be properly formatted and
  comply with standard transaction rules.
- **Financial Requirements**: Users must have enough funds to cover:
  - Pre-execution checks (including signature validation and transaction type
    checks).
  - Transaction execution costs.
  - A bond required for sequencer registration.
