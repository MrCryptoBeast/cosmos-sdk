# ADR 048: SIGN_MODE_TEXTUAL

## Changelog

- Dec 06, 2021: Initial Draft

## Status

Draft

## Abstract

This ADR specifies SIGN_MODE_TEXTUAL, a new string-based sign mode that is targetted at signing with hardware devices.

## Context

Protobuf-based SIGN_MODE_DIRECT has been introduced in [ADR-020](./adr-020-protobuf-transaction-encoding.md) and is intended to replace SIGN_MODE_LEGACY_AMINO_JSON in most situations, such as mobile wallets and CLI keyrings. However, the [Ledger](https://www.ledger.com/) hardware wallet is still using SIGN_MODE_LEGACY_AMINO_JSON for displaying the sign bytes to the user. Hardware wallets cannot transition to SIGN_MODE_DIRECT as it is binary-based and thus not suitable for display to end-users.

In an effort to remove Amino from the SDK, a new sign mode needs to be created for hardware devices. [Initial discussions](https://github.com/cosmos/cosmos-sdk/issues/6513) propose a string-based sign mode, which this ADR formally specifies.

## Decision

We propose to have SIGN_MODE_TEXTUAL’s signing payload `SignDocTextual` to be an array of strings. Each string would correspond to one "screen" on the hardware wallet device, with no (or little, TBD) additional formatting done by the hardware wallet app itself.

```proto
message SignDocTextual {
  repeated string screens = 1;
}
```

The string array MUST follow the specifications below.

### 1. Bijectivity with Protobuf transactions

The encoding and decoding operations between a Protobuf transaction (whose definition can be found [here](https://github.com/cosmos/cosmos-sdk/blob/master/proto/cosmos/tx/v1beta1/tx.proto#L13)) and the string array MUST be bijective.

We concede that bijectivity is not strictly needed. Avoiding transaction malleability only requires collision resistance on the encoding. Lossless encoding also does not require decodability. However, bijectivity assures both non-malleability and losslessness.

This also prevents users signing over hashed transaction metadata, which is a security concern for Ledger (the company).

We propose to maintain functional tests using bijectivity in the SDK to assure losslessness and no malleability.

### 2. Only ASCII 32-127 characters allowed

Ledger devices have limited character display capabilities, so all strings MUST only contain ASCII characters in the 32-127 range.

In particular, the newline `"\n"` (ASCII: 10) character is forbidden.

### 3. All strings have the `<key>: <value>` format

All strings MUST match the following Regex: `TODO`.

This is helpful for UIs displaying SignDocTextual to users. This MAY be used in the Ledger app to perform custom on-screen formatting, for example to break long lines into multiple screens.

The `<value>` itself can contain the `": "` characters.

### 4. Values are encoded using Value Renderers

Value Renderers describe how values of different types are rendered in the string array. The full specification of Value Renderers can be found in [Annex 1](./adr-048-sign-mode-textual-annex1.md).

### 5. Strings starting with `*` are only shown in Expert mode

Ledger devices have the an Expert mode for advanced users. Strings starting with the `*` character will only be shown in Expert mode.

### 6. The string array format

Below is the general format of a TX with N msgs. Each new line corresponds to a new screen on the Ledger device. `//` denotes comments and are not shown on the Ledger device.

### 7. Encoding of the Transaction Envelope

We define "transaction envelope" as all data in a transaction that is not in the `TxBody`. Transaction envelope includes fee, signer infos and memo, but don't include `Msg`s.

```
Chain ID: <string>
Account number: <uint64>
Sequence: <uint64>
<TxBody>                          // See 8.
Fee: <coins>
*Fee payer: <string>              // Skipped if no fee_payer set
*Fee granter: <string>            // Skipped if no fee_granter set
Memo: <string>                    // Skipped if no memo set
*Gas Limit: <uint64>
*Timeout Height:  <uint64>        // Skipped if no timeout_height set
Tipper: cosmos1ghi...ghi          // If there's a tip
Tip: 1.0 atom
*Signers:
*Signer (1/3):
*Public Key:                      // base64-encoded pk or hex TBD
*Sign mode: DIRECT
*Signer (2/3):
// --snip--
End of signers
```

### Wire Format

This string array is encoded as a single `\n`-delimited string before transmitted to the hardware device.

### Examples

#### Simple MsgSend

JSON:

```json
{
  "body": {
    "messages": [
      {
        "@type": "/cosmos.bank.v1beta1.MsgSend",
        "from": "cosmos1abc...abc",
        "to": "cosmos1def...def",
        "amount": [
          {
            "denom": "uregen",
            "amount": 10000000
          }
        ]
      }
    ]
  },
  "auth_info": {
    "signer_infos": [
      {
        "public_key": "iQ==",
        "mode_info": { "single": { "mode": "SIGN_MODE_TEXTUAL" } },
        "sequence": 2
      }
    ],
    "fee": {
      "amount": [
        {
          "denom": "regen",
          "amount": 0.002
        }
      ],
      "gas_limit": 100000
    }
  },
  // Additional data used in sign doc.
  "chain_id": "regen-1",
  "account_number": 10
}
```

SIGN_MODE_TEXTUAL:

```
Chain ID: regen-1
Account number: 10
Sequence: 2
This transaction has 1 message:
Message (1/1): bank send coins
from: cosmos1abc...abc
to: cosmos1def...def
amount: 10 regen            // Conversion from uregen to regen using value renderers
End of messages
Fee: 0.002 regen
*Gas: 100,000
This transaction has 1 signer:
```

## Consequences

### Backwards Compatibility

### Positive

### Negative

### Neutral

## Further Discussions

While an ADR is in the DRAFT or PROPOSED stage, this section should contain a summary of issues to be solved in future iterations (usually referencing comments from a pull-request discussion).
Later, this section can optionally list ideas or improvements the author or reviewers found during the analysis of this ADR.

## References

- Initial discussion: https://github.com/cosmos/cosmos-sdk/issues/6513