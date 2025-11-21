# IOU -> Different IOU

**Setup:**

Issuer has `rippling` flag set.
Issuer has trust lines for AAA to Alice and Market Maker.
Issuer has trust lines for AAB to Bob and Market Maker.
Issuer issues 100 AAA to Alice.
Issuer issues 10000 AAB to Market Maker.

Market Maker creates an offer:
- takerGets: 10000 AAB
- takerPays: 10000 AAA

**Path finding:**

Not used

**Payment:**

Alice sends payment:
- Destination: Bob
- SendMax: 10 AAA
- Amount: 5 AAB
- Paths: None
- No flags

**Default path:**

- Normalized path:
  - Type=0x31: Account: Alice, currency: AAA, issuer: Alice
  - Type=0x1: Account: Issuer
  - Type=0x30: Order book: AAA/AAB
  - Type=0x1: Account: Issuer
  - Type=0x1: Account: Bob

**Convert to strands**:

- Default path becomes:
  - DirectIPaymentStep: Alice -> Issuer
  - BookStep: AAA/MPT
  - DirectIPaymentStep: Issuer -> Bob


# IOU -> MPT

**Setup:**

Issuer has `rippling` flag set.
Issuer has trust lines for AAA to Alice and Market Maker.
Issuer issues 100 AAA to Alice.

Issuer creates an MPT Issuance, with flags: tfMPTCanTrade, tfMPTCanTransfer. 
Issuer sends 1000 MPT to Market Maker.

Market Maker creates an offer:
- takerGets: 100 MPT
- takerPays: 100 AAA

**Path finding:**

Not used

**Payment:**

Alice sends payment:
- Destination: Bob
- SendMax: 10 AAA
- Amount: 5 MPT
- Paths: None
- No flags

**Default path:**

- Normalized path:
  - Type=0x31: Account: Alice, currency: AAA, issuer: Alice
  - Type=0x1: Account: Issuer
  - Type=0x60: Order book: AAA/MPT
  - Type=0x1: Account: Issuer
  - Type=0x1: Account: Bob

**Convert to strands**:

- Default path becomes:
  - DirectIPaymentStep: Alice -> Issuer
  - BookStep: AAA/MPT
  - MPTEndpointPaymentStep: Issuer -> Bob


