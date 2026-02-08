# Index

- [1. Introduction](#1-introduction)
- [2. Ledger Entries](#2-ledger-entries)
    - [2.1. AccountRoot Ledger Entry](#21-accountroot-ledger-entry)
        - [2.1.1. Object Identifier](#211-object-identifier)
        - [2.1.2. Fields](#212-fields)
            - [2.1.2.1. Flags](#2121-flags)
        - [2.1.3. Reserves](#213-reserves)
    - [2.2. RippleState Ledger Entry](#22-ripplestate-ledger-entry)
    - [2.3. MPT Ledger Entries](#23-mpt-ledger-entries)
    - [2.4. AMM Ledger Entries](#24-amm-ledger-entries)
- [3. Transactions](#3-transactions)
    - [3.1. Payment Transaction](#31-payment-transaction)
        - [3.1.1. Failure Conditions](#311-failure-conditions)
        - [3.1.2. State Changes](#312-state-changes)
- [4. Payment Execution Paths](#4-payment-execution-paths)
    - [4.1. Direct XRP Payment Execution](#41-direct-xrp-payment-execution)
    - [4.2. Cross-Currency Payment Execution](#42-cross-currency-payment-execution)
        - [4.2.1. Path Finding](#421-path-finding)
        - [4.2.2. Flow Execution](#422-flow-execution)

# 1. Introduction

Payments are the fundamental mechanism for transferring value on the XRP Ledger. They enable the movement of XRP,
IOUs, and Multi-Purpose Tokens (MPTs) between accounts, either directly or through intermediary paths that
leverage trust lines and order books.

A Payment transaction can operate in different modes:

1. [Direct XRP Payment](#41-direct-xrp-payment-execution): Simple transfer of XRP from one account to another
2. [Cross-Currency Payment](#42-cross-currency-payment-execution): Converting one currency to another through intermediary steps, using paths through the decentralized exchange. With the [MPTokensV2](https://xrpl.org/resources/known-amendments#mptokensv2) amendment, cross-currency payments support all combinations of XRP, tokens, and MPTs.
3. Since [MPTokensV2](https://xrpl.org/resources/known-amendments#mptokensv2), MPT->MPT payments are using the cross-currency payment execution. Please see [MPT Payment Execution](../mpts/README.md#4-mpt-payment-execution) for a description of MPT transfer mechanics. Before it was moved to cross-currency payment system, Payment transactor completed the steps as outlined in that document manually.   

The payment system uses the path finding algorithm and the Payment Engine to discover the most efficient routes for cross-currency transactions,
automatically handling currency conversion, fees, and liquidity constraints. Payments can optionally specify a `DomainID` field; when specified, the payment will consume offers only from that domain's order book (domain offers and hybrid offers within that domain). Without a `DomainID`, payments consume only from the open order book (open offers and hybrid offers). See [Domain and Hybrid Offers](../offers/README.md#15-permissioned-dex) and [PermissionedDomains documentation](../permissioned_domains/README.md) for details.

Payments execute either fully or partially (when `tfPartialPayment` flag is set), or fail with an error code if insufficient liquidity exists.

# 2. Ledger Entries

The Payment transaction interacts with different ledger entry types depending on the payment mode:

- **Direct XRP payments**: Modify only `AccountRoot` entries to update XRP balances
- **Direct MPT payments**: Modify `MPToken` entries (sender and receiver balances), and `MPTokenIssuance` entry (to track outstanding amount and transfer fee burns)
- **Cross-currency payments**: May modify `AccountRoot`, `RippleState` (trust lines), `DirectoryNode` (owner directories and order book directories), `Offer` (consuming or partially consuming offers from the order book), `AMM` (when AMM liquidity is used for swaps), `MPTokenIssuance` entry (to track outstanding amount), and `MPToken` entries for sender and receiver holders

## 2.1. AccountRoot Ledger Entry

The `AccountRoot` ledger entry represents an account on the XRP Ledger and stores the account's XRP balance, sequence
number, and various flags that control account behavior.

### 2.1.1. Object Identifier

The key of the `AccountRoot` object is the result
of [SHA512-Half](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes) of the following values
concatenated in order:

- The `AccountRoot` space key `0x0061` (lowercase `a`)
- The `AccountID` of the account.

### 2.1.2. Fields

Please
see [AccountRoot Fields](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/accountroot#accountroot-fields)

#### 2.1.2.1. Flags

Please
see [AccountRoot Flags](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/accountroot#accountroot-flags)

Key flags relevant to payments:

- `lsfRequireDestTag`: Requires incoming payments to include a destination tag
- `lsfDepositAuth`: Requires authorization for incoming payments (except XRP under specific conditions)
- `lsfPasswordSpent`: Tracks whether the account has used its one-time
  free [SetRegularKey](https://xrpl.org/docs/references/protocol/transactions/types/setregularkey) transaction. Cleared
  when the account receives a payment to allow another free regular key change

### 2.1.3. Reserves

See [Reserves](https://xrpl.org/docs/concepts/accounts/reserves) for complete details on reserve requirements.

**Payment-specific reserve behavior:**

When a Payment transaction creates a new destination account (destination does not exist), the payment must deliver at
least the base reserve amount in XRP. If the XRP amount is below the base reserve, the payment fails with
`tecNO_DST_INSUF_XRP`.

## 2.2. RippleState Ledger Entry

See [Trust Lines Documentation](../trust_lines/README.md#21-ripplestate-ledger-entry) for complete details on
`RippleState` ledger entry.

## 2.3. MPT Ledger Entries

See [MPT Documentation](../mpts/README.md#2-ledger-entries) for complete details on MPT related ledger entries.

## 2.4. AMM Ledger Entries

See [AMM Documentation](../amms/README.md#2-ledger-entries) for complete details on AMM related ledger entries.


# 3. Transactions

## 3.1. Payment Transaction

The Payment transaction transfers value from one account to another, supporting [XRP](../glossary.md#xrp), [IOUs](../glossary.md#iou), and [MPTs](../glossary.md#mpt). 
It can operate as a simple direct transfer or use pathfinding for cross-currency conversions.

Fields are described
in [Payment Fields](https://xrpl.org/docs/references/protocol/transactions/types/payment#payment-fields)

Flags are described
in [Payment Flags](https://xrpl.org/docs/references/protocol/transactions/types/payment#payment-flags)

**Automatic Pathfinding with `build_path`**

The `sign` and `submit` RPC commands accept a `build_path` parameter that triggers automatic pathfinding before signing/submitting the transaction:

```json
{
  "command": "sign",
  "tx_json": {
    "TransactionType": "Payment",
    "Account": "rSource...",
    "Destination": "rDest...",
    "Amount": {
      "currency": "USD",
      "value": "100",
      "issuer": "rGateway..."
    }
  },
  "build_path": true
}
```

When `build_path` is `true`:

- [Pathfinding](../path_finding/README.md) runs automatically using the `path_search_old` configuration value
- Up to 4 best paths are found and inserted into the `Paths` field (this is hardcoded in `TransactionSign.cpp`)
- Only works if `Paths` is not already specified
- Cannot be used for XRP-to-XRP payments

### 3.1.1. Failure Conditions

**Static validation**[^static-validation]

[^static-validation]: Static validation (preflight): [`checkExtraFeatures`](https://github.com/gregtatcam/rippled/blob/a72c3438eb0591a76ac829305fcbcd0ed3b8c325/src/xrpld/app/tx/detail/Payment.cpp#L53-L62), [`getFlagsMask`](https://github.com/gregtatcam/rippled/blob/a72c3438eb0591a76ac829305fcbcd0ed3b8c325/src/xrpld/app/tx/detail/Payment.cpp#L66-L77), [`preflight`](https://github.com/gregtatcam/rippled/blob/a72c3438eb0591a76ac829305fcbcd0ed3b8c325/src/xrpld/app/tx/detail/Payment.cpp#L81-L245)

Assuming [MPTokensV1](https://xrpl.org/resources/known-amendments#mptokensv1) and [MPTokensV2](https://xrpl.org/resources/known-amendments#mptokensv2) is enabled:

- `temDISABLED`:
    - transaction contains `sfCredentialIDs` and [Credentials](https://xrpl.org/resources/known-amendments#credentials) amendment is not enabled.
    - transaction contains `sfDomainID` and [PermissionedDEX](https://xrpl.org/resources/known-amendments#permissioneddex) is not enabled.
- `temINVALID_FLAG`: transaction flags contain invalid flags for the payment type.
- `temMALFORMED`:
    - `sfCredentialIDs` array is empty or exceeds maximum size of 8. To leave credential IDs out, leave out the entire field.
    - `sfCredentialIDs` array contains duplicate credential IDs
- `temBAD_AMOUNT`:
    - `Amount` is XRP and mantissa is bigger than `100000000000000000ull`.
    - `SendMax` is XRP and mantissa is bigger than `100000000000000000ull`.[^isLegalNet-sendmax]
    - `SendMax` is specified but is negative or zero.
    - `Amount` amount is negative or zero.
    - `DeliverMin` is specified without `tfPartialPayment`.
    - `DeliverMin` is XRP and mantissa is bigger than `100000000000000000ull`, or `DeliverMin` is negative or zero.[^delivermin-checks]
    - `DeliverMin` currency is different from `Amount` currency.
    - `DeliverMin` is greater than `Amount`.
- `temBAD_CURRENCY`: either `Amount` or `SendMax` (or the implied source amount) is a non-native asset that uses the XRP currency code.
- `temDST_NEEDED`: destination account is not specified.
- `temREDUNDANT`: payment is from account to itself with same currency and no `Paths` field. `Paths` are required because perhaps they will allow arbitrage.
- `temBAD_SEND_XRP_MAX`: XRP->XRP payment specifies `SendMax`.
- `temBAD_SEND_XRP_PATHS`: XRP->XRP payment specifies `Paths`.
- `temBAD_SEND_XRP_PARTIAL`: XRP->XRP payment has `tfPartialPayment` flag.
- `temBAD_SEND_XRP_LIMIT`: XRP->XRP payment has `tfLimitQuality` flag, or MPT->MPT payment has `tfLimitQuality` flag (only if [MPTokensV2](https://xrpl.org/resources/known-amendments#mptokensv2) amendment is not enabled).
- `temBAD_SEND_XRP_NO_DIRECT`: XRP->XRP payment has `tfNoRippleDirect` flag, or MPT->MPT payment has `tfNoRippleDirect` flag (only if [MPTokensV2](https://xrpl.org/resources/known-amendments#mptokensv2) amendment is not enabled).

**Validation against the ledger view**[^preclaim-validation]

[^preclaim-validation]: Validation against ledger view (preclaim): [`checkPermission`](https://github.com/gregtatcam/rippled/blob/a72c3438eb0591a76ac829305fcbcd0ed3b8c325/src/xrpld/app/tx/detail/Payment.cpp#L249-L285), [`preclaim`](https://github.com/gregtatcam/rippled/blob/a72c3438eb0591a76ac829305fcbcd0ed3b8c325/src/xrpld/app/tx/detail/Payment.cpp#L288-L385)
[^isLegalNet-sendmax]: Both Amount and SendMax checked via isLegalNet: [`Payment.cpp`](https://github.com/gregtatcam/rippled/blob/a72c3438eb0591a76ac829305fcbcd0ed3b8c325/src/xrpld/app/tx/detail/Payment.cpp#L126)
[^delivermin-checks]: DeliverMin checked for legal amount and positive value: [`Payment.cpp`](https://github.com/gregtatcam/rippled/blob/a72c3438eb0591a76ac829305fcbcd0ed3b8c325/src/xrpld/app/tx/detail/Payment.cpp#L216-L238)

- Destination account does not exist:
    - `tecNO_DST`: payment is not XRP
    - `telNO_DST_PARTIAL`: `tfPartialPayment` flag is set. User cannot fund a new account with a partial payment.
    - `tecNO_DST_INSUF_XRP`: XRP amount is below reserve.
- `tecDST_TAG_NEEDED` : destination account has `lsfRequireDestTag` flag set and transaction did not specify `DestinationTag` field.
- `telBAD_PATH_COUNT`:
    - the `Paths` field contains more than 6 paths.
    - any `Path` in `Paths` has more than 8 elements.
- `tecBAD_CREDENTIALS`: Credential validation failed:
    - Any credential ID in `sfCredentialIDs` doesn't exist in the ledger
    - Any credential doesn't belong to the source account
    - Any credential isn't accepted (missing `lsfAccepted` flag)
- `tecNO_PERMISSION`: `DomainID` is present and either sending or receiving account is not in Domain (not the domain owner and does not hold a valid, non-expired accepted credential).
- `terNO_DELEGATE_PERMISSION`: Transaction specifies a delegate but:
    - The delegate authorization doesn't exist in the ledger
    - The delegate doesn't have transaction-level permission for Payment
    - For granular permissions: the payment is not a direct payment (has `Paths` or `SendMax` with different asset), OR
    - For granular permissions: neither PaymentMint (when source is issuer) nor PaymentBurn (when destination is issuer) permission is granted

**Validation during doApply**

**Common validations (all payment types):**

- `tecNO_PERMISSION`: For cross-currency payments, if DepositPreauth amendment is not enabled and destination has `lsfDepositAuth` flag

**Direct XRP Payments:**[^direct-xrp-payment]

[^direct-xrp-payment]: Direct XRP payment execution: [`Payment.cpp`](https://github.com/gregtatcam/rippled/blob/a72c3438eb0591a76ac829305fcbcd0ed3b8c325/src/xrpld/app/tx/detail/Payment.cpp#L607-L698)

- `tefINTERNAL`: Source account does not exist.
- `tecUNFUNDED_PAYMENT`: Source account cannot send the payment because doing it would leave it with insufficient funds
  to cover reserve and pay the fee.
- `tecNO_PERMISSION`: Destination is a pseudo-account.
- If [DepositAuth](https://xrpl.org/resources/known-amendments#depositauth) amendment is enabled and destination has
  `lsfDepositAuth` flag:
    - Payment succeeds if source == destination (paying yourself)
    - Payment succeeds if destination balance <= base reserve AND payment amount <= base reserve (prevents account
      wedging)
    - Otherwise, deposit preauthorization is verified and may fail with:
        - `tecEXPIRED`: Any credential in `sfCredentialIDs` is expired
        - `tecNO_PERMISSION`: Source is not deposit preauthorized by destination (either by account or by credentials)

**Cross-Currency Payments:**

- If DepositPreauth amendment is enabled and destination has `lsfDepositAuth` flag, deposit preauthorization is verified
  and may fail with:
    - `tecEXPIRED`: Any credential in `sfCredentialIDs` is expired
    - `tecNO_PERMISSION`: Source is not deposit preauthorized by destination (either by account or by credentials)
- RippleCalc, which is a thin wrapper that calls the [Flow engine](../flow/README.md), is invoked to execute the provided payment paths. Flow may fail during path conversion or execution. See [Flow Validation and Error Codes](../flow/README.md#8-validation-and-error-codes) for complete details on error codes that can be returned during cross-currency payment execution.
- `tecPATH_PARTIAL`: `DeliverMin` was specified and the delivered amount is less than `DeliverMin`

### 3.1.2. State Changes

**XRP Payments:**

- `AccountRoot` object is **modified**:
    - Source account: Balance decreased by payment amount
    - Destination account (if exists): Balance increased by delivered amount
    - Destination account (if exists and has `lsfPasswordSpent` flag): Clear the flag

- `AccountRoot` object is **created**:
    - When the destination account does not exist
    - Fields set:
        - `Account`: Destination account ID
        - `Balance`: Payment amount
        - `Sequence`: Current ledger sequence if DeletableAccounts amendment is enabled, otherwise 1

**Cross-Currency Payments:**

State changes for cross-currency payments depend on the execution path determined by RippleCalc/Flow.
See [Flow documentation](../flow/README.md) for detailed mechanics of path execution.

- `AccountRoot` objects are **modified**:
    - Source account: Balance decreased by actual amount consumed
    - Destination account: Balance increased by delivered amount
    - Destination account (if has `lsfPasswordSpent` flag): Clear the flag
    - Intermediate accounts in payment path: Balances adjusted according to path execution

- `RippleState` objects are **modified**:
    - Trust lines along the payment path: Balances updated according to transfers
    - Trust line quality and flags may affect transfer amounts

- `Offer` objects may be **modified** or **deleted**:
    - When: BookStep consumes order book liquidity
    - Modified: Offer is partially consumed, amounts reduced
    - Deleted: Offer is fully consumed or becomes unfunded

- `DirectoryNode` objects may be **modified**:
    - When: Offers are consumed and removed from order book directories
    - Directory entries updated to reflect consumed offers

- `AMM` objects may be **modified**:
    - When: AMM liquidity is used for currency conversion
    - AMM pool balances adjusted according to swap amounts

**MPT Payments:**

- Source's `MPToken` is **modified** (if source is not the issuer):
  - `MPTAmount`: Decreased by sent amount (including transfer fee if applicable)

- Destination's `MPToken` is **modified** (if destination is not the issuer):
  - `MPTAmount`: Increased by received amount

- Transfer fee effect (when neither source nor destination is the issuer):
  - `MPTokenIssuance` `OutstandingAmount`: Decreased by transfer fee amount
  - The fee is effectively burned from circulation, reducing total supply

- `MPTokenIssuance` is **modified**:
  - When source is issuer: `OutstandingAmount` increased by sent amount (issuer minting tokens)
  - When destination is issuer: `OutstandingAmount` decreased by received amount (tokens burned, removed from circulation)
  - When neither is issuer: `OutstandingAmount` decreased by transfer fee amount (if transfer fee applies; unchanged if no fee or fee is waived, e.g., clawback)

**Important**:
- The receiver must already have an `MPToken` entry before receiving a payment (unless the receiver is the issuer). If the receiver has no `MPToken` and is not the issuer, the payment fails with `tecNO_AUTH`. Holders create their `MPToken` entries using the `MPTokenAuthorize` transaction.
- The issuer never has an `MPToken` entry and cannot hold a balance of their own issuance. When MPTs are sent to the issuer, they are burned from circulation by decreasing `OutstandingAmount`.

# 4. Payment Execution Paths

All payment types are processed through the Payment transaction but follow different execution paths based on the
currencies and parameters involved:

- **Direct XRP payments**: Execute simple balance transfers when both `Amount` and `SendMax` (if present) are XRP and no `Paths` are specified
- **Cross-currency payments**: Invoke the [Flow engine](../flow/README.md) when `SendMax` is specified, `Paths` are provided, or `Amount` is a non-XRP asset.
- **Direct MPT payments**: Execute MPToken transfers when `Amount` holds an MPT issue and MPTokensV2 amendment is not enabled

The Payment transaction determines which path to take during the `doApply` phase based on these conditions.

## 4.1. Direct XRP Payment Execution

Direct XRP payments are the simplest payment type, transferring XRP directly from one account to another without any intermediate steps or currency conversion. The execution varies based on whether the destination account exists.

**When the destination account exists**: The payment decreases the source account's `Balance` by the payment amount and increases the destination account's `Balance` by the same amount. If the destination account has the `lsfPasswordSpent` flag set, it is cleared to allow another free `SetRegularKey` transaction.

**When the destination account does not exist**: A new `AccountRoot` entry is created for the destination with the payment amount as its initial balance. The account's `Sequence` is set to the current ledger sequence (if [DeletableAccounts](https://xrpl.org/resources/known-amendments#deletableaccounts) is enabled) or 1 otherwise. The payment must meet the base reserve requirement (see [Reserves](#213-reserves)), or it fails with `tecNO_DST_INSUF_XRP`.

All validation checks-including reserve requirements, deposit authorization, and destination tags-are performed before execution. See [Failure Conditions](#311-failure-conditions) for complete validation rules.

## 4.2. Cross-Currency Payment Execution

Cross-currency payments convert one currency to another through intermediary steps, leveraging MPTs, trust lines, order books, and AMM liquidity. These payments are executed when the payment involves non-XRP currencies, specifies `SendMax`, or includes `Paths`.

Cross-currency payment execution involves two complementary components: path finding and flow execution.

### 4.2.1. Path Finding

Before a payment can execute, viable routes from source to destination must be discovered. This is handled by the [Path Finding Protocol](../path_finding/README.md), which searches the ledger graph to find potential paths through trust lines (for tokens), order books (for XRP, tokens, and MPTs), and AMM pools (for XRP, tokens, and MPTs).

Path finding can be initiated in three ways:

- **User-specified paths**: Clients can provide explicit `Paths` in the Payment transaction, bypassing path finding entirely
- **RPC path finding**: Clients use RPC endpoints to discover paths before submitting the transaction:
  - [`path_find`](../path_finding/README.md#72-path_find-rpc) - Modern interface with streaming subscriptions (`path_find create`, `path_find status`, `path_find close`)
  - [`ripple_path_find`](../path_finding/README.md#71-ripple_path_find-rpc-legacy) - Legacy one-shot path finding (deprecated but still supported)
- **Default paths**: When no paths are specified, the payment engine automatically generates simple default paths (direct trust line to issuer, XRP-bridged paths)

The RPC endpoints accept parameters like `source_account`, `destination_account`, `destination_amount`, and optionally `source_currencies` (to constrain which currencies the pathfinder considers as potential sources). For each source currency, the pathfinder creates a separate search instance, discovers paths through the ledger graph, simulates each path to measure quality and liquidity, ranks paths by quality, and returns the best paths to the client.

### 4.2.2. Flow Execution

Once paths are available (whether from RPC, user-specified, or default paths), the Payment transaction executes by delegating to the [Flow engine](../flow/README.md). Flow converts the paths into **strands** - sequences of executable **steps** that move value:

- **DirectStepI**: Transfers tokens between accounts via trust lines
- **BookStep**: Converts currencies through order books and AMM pools
- **XRPEndpointStep**: Transfers XRP to/from accounts
- **MPTEndpointStep**: Transfers MPTs to/from accounts

Flow [ranks strands by quality](../flow/README.md#62-qualityupperbound) (exchange rate) and [iteratively consumes liquidity](../flow/README.md#6-iterative-strands-evaluation-strandsflow) from the best available strands, dynamically re-ranking as their quality changes, until the payment amount is satisfied or all liquidity is exhausted. Each strand evaluation uses a [two-pass algorithm](../flow/README.md#7-single-strand-evaluation-strandflow): a reverse pass works backwards from the desired output to calculate required input, then a forward pass verifies and executes the actual transfer.

See [Flow documentation](../flow/README.md) for detailed mechanics of path execution, strand evaluation, and step-by-step state changes.