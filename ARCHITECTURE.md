# StellarVault - Multi-Signature Treasury System Architecture

## Overview

StellarVault is a decentralized, multi-signature treasury system built on Stellar using Soroban smart contracts. It enables secure, governance-based control over transactions through collaboration between authorized signers. The system implements multi-signature approval with timelocked execution, ensuring no single signer can independently execute transactions.

**Core Principles:**

- Security: Cryptographic verification and reentrancy protection
- Efficiency: Optimized for Stellar's consensus model
- Role-Based Access Control: Granular permission management
- Timelocked Execution: Two-layer security through quorum + delay

---

## System Architecture

### High-Level Flow

```
User/Admin
    ↓
AccessRoles Contract (Role Management)
    ↓
TransactionProposal Contract (Workflow Management)
    ↓
Treasury Contract (Asset Management)
    ↓
DelayTime Contract (Temporal Constraints)
```

### Contract Interactions

1. **Admin** sets up system via AccessRoles
2. **Proposer** creates transaction proposal via TransactionProposal
3. **Signers** confirm proposal via TransactionProposal
4. **DelayTime** enforces waiting period
5. **Treasury** executes token transfers upon approval

---

## Module Separation

### 1. AccessRoles Contract

**Purpose:** Manages role-based permissions and access control for the treasury system.

**Roles:**

- `ProposalCreated { proposal_id: u64, proposer: AccountId, amount: i128 }`
- `ConfirmationRecorded { proposal_id: u64, signer: AccountId }`
- `ProposalConfirmed { proposal_id: u64, confirmations: u32 }`
- `ProposalExecuted { proposal_id: u64, recipient: AccountId, amount: i128 }`
- `ProposalCancelled { proposal_id: u64 }`
- `FeeUpdated { new_fee: i128 }`
  **Key State Variables:**

| Variable       | Type | Description                                          |
| -------------- | ---- | ---------------------------------------------------- |
| `quorum`       | u32  | Minimum signers required for approval (e.g., 2 of 3) |
| `totalSigners` | u32  | Total number of accounts with Signer_Role            |
| `signers`      | Map  | Mapping of signer accounts to role status            |

**Core Functions:**

**Core Functions:**

- `add_cosigner(admin: AccountId, signer: AccountId) -> Result`
  - Only callable by Default_Admin_Role
  - Checks if address already has Signer_Role
  - Grants Signer_Role if not already assigned
  - Increments totalSigners
  - Emits event: SignerAdded

- `remove_signer(admin: AccountId, signer: AccountId) -> Result`
  - Only callable by Default_Admin_Role
  - Revokes Signer_Role
  - Decrements totalSigners
  - Validates quorum <= totalSigners after removal
  - Emits event: SignerRemoved

- `set_quorum(admin: AccountId, new_quorum: u32) -> Result`
  - Only callable by Default_Admin_Role
  - Validates new_quorum <= totalSigners
  - Validates new_quorum > 0
  - Emits event: QuorumChanged

- `has_role(account: AccountId, role: BytesN(32)) -> bool`
  - Public function
  - Returns true if address has the specified role
  - Used by other contracts for permission checks

- `get_default_admin_role() -> BytesN(32)`
  - Returns the hash identifier of Default_Admin_Role

- `get_signer_role() -> BytesN(32)`
  - Returns the hash identifier of Signer_Role

- `get_quorum() -> u32`
  - Returns current quorum requirement

- `get_total_signers() -> u32`
  - Returns total count of active signers

**Security Considerations:**

- Only explicitly granted roles can modify system parameters
- Quorum cannot exceed total signers
- Cannot remove signers below quorum threshold

---

### 2. TransactionProposal Contract

**Purpose:** Manages the complete lifecycle of transaction proposals through a multi-signature workflow.

**Key Concepts:**

- **Proposal Fee:** Paid in native Stellar tokens (stroops) to discourage spam
- **Confirmation Voting:** Signers vote independently to approve proposals
- **Delay Phase:** Enforced waiting period between approval and execution
- **State Transitions:** Pending → Confirmed → Executed/Cancelled

**Proposal Lifecycle:**

```
CREATE (fee paid)
    ↓
PENDING (awaiting signer confirmations)
    ↓
CONFIRMED (quorum reached, enters delay)
    ↓
DELAY PHASE (time lock enforced)
    ↓
EXECUTED (if delay passed & quorum still met)
    or
CANCELLED (if quorum reached to cancel after delay)
```

**Key State Variables:**

| Variable           | Type | Description                               |
| ------------------ | ---- | ----------------------------------------- |
| `proposal_fee`     | i128 | Fee in stroops for creating proposals     |
| `proposals`        | Map  | Storage of all proposals                  |
| `confirmations`    | Map  | Map of proposal_id → signer confirmations |
| `proposal_counter` | u64  | Auto-incrementing proposal ID             |

**Proposal Structure:**

```
Proposal {
    id: u64,
  proposer: AccountId,
  target_contract: AccountId,
    function_name: String,
    params: Vec<Val>,
  recipient: AccountId,
    amount: i128,
    confirmations: u32,
    executed: bool,
    cancelled: bool,
    created_at: u64,
    delay_until: u64,
    state: ProposalState, // PENDING, CONFIRMED, EXECUTED, CANCELLED
}
```

**Core Functions:**

- `create_proposal(proposer: AccountId, recipient: AccountId, amount: i128, description: String) -> Result<u64>`
  - Proposer must have Signer_Role
  - Requires proposal_fee payment
  - Creates new proposal with PENDING state
  - Returns proposal ID
  - Emits event: ProposalCreated

- `confirm_proposal(signer: AccountId, proposal_id: u64) -> Result`
  - Only callable by address with Signer_Role
  - Signer must not have already confirmed this proposal
  - Records signer confirmation
  - If confirmations >= quorum:
    - Sets state to CONFIRMED
    - Sets delay_until = now + execution_delay
    - Emits event: ProposalConfirmed
  - Emits event: ConfirmationRecorded

- `execute_proposal(executor: AccountId, proposal_id: u64) -> Result`
  - Proposal must be in CONFIRMED state
  - Current time must be >= delay_until
  - Calls Treasury.withdraw_for_proposal()
  - Sets state to EXECUTED
  - Emits event: ProposalExecuted
  - Emits event: TokenTransferred

- `cancel_proposal(signer: AccountId, proposal_id: u64) -> Result`
  - Proposal must be in CONFIRMED state
  - Current time must be > delay_until (after delay has passed)
  - Requires quorum confirmations for cancellation
  - Sets state to CANCELLED
  - Returns proposal_fee to proposer
  - Emits event: ProposalCancelled

- `get_proposal(proposal_id: u64) -> Proposal`
  - Public read function
  - Returns proposal details

- `get_proposal_confirmations(proposal_id: u64) -> u32`
  - Public read function
  - Returns number of confirmations received

- `set_proposal_fee(admin: AccountId, new_fee: i128) -> Result`
  - Only callable by Default_Admin_Role
  - Updates proposal_fee
  - Emits event: FeeUpdated

- `get_proposal_fee() -> i128`
  - Returns current proposal fee

**Events:**

- `ProposalCreated { proposal_id: u64, proposer: Address, amount: i128 }`
- `ConfirmationRecorded { proposal_id: u64, signer: Address }`
- `ProposalConfirmed { proposal_id: u64, confirmations: u32 }`
- `ProposalExecuted { proposal_id: u64, recipient: Address, amount: i128 }`
- `ProposalCancelled { proposal_id: u64 }`
- `FeeUpdated { new_fee: i128 }`
- `ProposalCreated { proposal_id: u64, proposer: AccountId, amount: i128 }`
- `ConfirmationRecorded { proposal_id: u64, signer: AccountId }`
- `ProposalConfirmed { proposal_id: u64, confirmations: u32 }`
- `ProposalExecuted { proposal_id: u64, recipient: AccountId, amount: i128 }`
- `ProposalCancelled { proposal_id: u64 }`
- `FeeUpdated { new_fee: i128 }`

**Security Considerations:**

- Proposal fee reduces spam and ensures cost for proposers
- Double-voting protection: a signer cannot confirm the same proposal twice
- Delay enforces mandatory waiting period
- Cancellation after delay prevents proposal abandonment
- Uses AccessRoles for permission checks

---

### 3. Treasury Contract

**Purpose:** Manages token reserves and executes authorized token transfers.

**Functions:**

- `deposit(depositor: AccountId, amount: i128) -> Result`
  - Records deposit in user balance
  - Smart contract must hold tokens
  - Validates amount > 0
  - Updates total reserves
  - Emits event: TokenDeposited

- `withdraw(withdrawer: AccountId, amount: i128) -> Result`
  - Only callable by the original depositor
  - Cannot withdraw more than user deposited
  - Validates amount > 0
  - Validates amount <= user_balance
  - Transfers tokens to withdrawer
  - Updates user balance and total reserves
  - Emits event: TokenWithdrawn

- `withdraw_for_proposal(proposal_id: u64, recipient: AccountId, amount: i128) -> Result`
  - Only callable by TransactionProposal contract
  - Validates proposal exists and is CONFIRMED
  - Validates delay period has passed
  - Validates treasury has sufficient balance
  - Transfers tokens to recipient
  - Updates treasury balance
  - Emits event: ProposalWithdrawal

`get_balance(account: AccountId) -> i128`

- Returns depositor's current balance

- `get_total_reserves() -> i128`
  - Returns total tokens held in treasury

- `get_treasury_info() -> TreasuryInfo`
  - Returns treasury state snapshot

**Events:**

- `TokenDeposited { depositor: AccountId, amount: i128 }`
- `TokenWithdrawn { withdrawer: AccountId, amount: i128 }`
- `ProposalWithdrawal { proposal_id: u64, recipient: AccountId, amount: i128 }`

**Security Considerations:**

- Only accepts calls from authorized TransactionProposal contract
- Maintains per-user deposit limits
- Validates sufficient reserve balance before transfers
- Prevents double-spending through indexed state

---

### 4. DelayTime Contract

**Purpose:** Implements and enforces temporal constraints on transaction execution.

**State Variables:**

| Variable          | Type | Description                                       |
| ----------------- | ---- | ------------------------------------------------- |
| `execution_delay` | u64  | Delay duration in seconds (e.g., 2 days = 172800) |

**Core Functions:**

- `set_execution_delay(admin: Address, delay_seconds: u64) -> Result`
  - Only callable by Default_Admin_Role
  - Updates execution_delay
  - Validates delay_seconds > 0
  - Validates delay_seconds <= MAX_DELAY (e.g., 30 days)
  - Emits event: DelayUpdated

- `get_execution_delay() -> u64`
  - Returns current execution delay in seconds

- `calculate_execution_time(proposal_created_at: u64) -> u64`
  - Pure calculation function
  - Returns: proposal_created_at + execution_delay
  - Used by TransactionProposal for time validation

- `is_delay_passed(proposal_created_at: u64, current_time: u64) -> bool`
  - Pure check function
  - Returns: current_time >= proposal_created_at + execution_delay

**Events:**

- `DelayUpdated { new_delay: u64 }`

**Security Considerations:**

- Prevents execution without mandatory delay
- Captures proposal timestamp for accurate delay calculation
- Bounded by MAX_DELAY to prevent extreme delays

---

## Security Boundaries

### Interface-Based Design

All contracts interact through well-defined interfaces:

```
I AccessRoles:
  - has_role(account: AccountId, role: BytesN(32)) -> bool
  - get_signer_role() -> BytesN(32)
  - get_quorum() -> u32

I TransactionProposal:
  - confirm_proposal(signer: AccountId, proposal_id: u64) -> Result
  - execute_proposal(executor: AccountId, proposal_id: u64) -> Result

I Treasury:
  - withdraw_for_proposal(proposal_id, recipient: AccountId, amount: i128) -> Result
  - get_balance(account: AccountId) -> i128

I DelayTime:
  - get_execution_delay() -> u64
  - is_delay_passed(created_at, current_time) -> bool
```

**Benefits:**

- Decouples contract implementations
- Enables contract upgrades without breaking dependencies
- Limits attack surface area
- Clear responsibility boundaries

### Multi-Layer Security Model

```
Layer 1: ROLE-BASED AUTHENTICATION
├─ Default_Admin_Role: Configuration only
├─ Signer_Role: Transaction participation
└─ No single role can execute unilaterally

Layer 2: PROPOSAL FEES
├─ Deters spam submissions
├─ Economically incentivizes good-faith proposals
└─ Fund recovery mechanism

Layer 3: QUORUM REQUIREMENT
├─ Requires minimum signer consensus
├─ Prevents majority-signer attacks
└─ Distributed authority

Layer 4: TIME LOCK
├─ Enforced delay between approval and execution
├─ Community response window
├─ Prevents rapid exploit attacks
└─ Enables governance intervention
```

### Specific Security Mitigations

| Threat                  | Mitigation                          |
| ----------------------- | ----------------------------------- |
| Single point of failure | Multi-signer requirement (quorum)   |
| Rapid exploit attacks   | Timelocked execution                |
| Reentrancy              | State checks before token transfers |
| Unauthorized access     | Role-based access control           |
| Spam proposals          | Proposal fee requirement            |
| Excessive delays        | MAX_DELAY bounded parameter         |
| Double-voting           | Per-signer confirmation tracking    |

---

## Stellar/Soroban Specific Considerations

### Ledger Architecture

- **Contracts deployed on:** Stellar testnet (initially), mainnet (production)
- **Storage:** Stellar ledger entries (contract data)
- **Token:** Native Stellar asset (stroops) for fees, treasury asset (configurable)
- **Time:** Unix timestamp from Stellar ledger

### Events and Logging

- Soroban events for proposal state changes
- Event indexing for offchain monitoring
- Admin dashboard tracking proposal history

### Gas/CPU Optimization

- Batch confirmations to reduce call overhead
- Indexed proposal lookups
- Efficient state storage layout

### Contract Deployment

```
1. Deploy AccessRoles
2. Deploy DelayTime
3. Deploy Treasury
4. Deploy TransactionProposal (refs to AccessRoles, DelayTime, Treasury)
5. Initialize system parameters (quorum, delay, fee)
6. Add initial signers
```

---

## Data Flow Diagrams

### Successful Proposal Execution Flow

```
Admin Creates System
    ↓
Admin Adds Signers (≥ quorum count)
    ↓
Proposer Creates Proposal (pays fee)
    ↓
Signer 1 Confirms → confirmations = 1
    ↓
Signer 2 Confirms → confirmations = 2 (quorum reached!)
    ↓
Proposal enters DELAY PHASE
    ↓
[Wait execution_delay seconds]
    ↓
Executor calls execute_proposal()
    ↓
Treasury transfers tokens to recipient
    ↓
Proposal marked EXECUTED
    ↓
Events emitted for offchain indexing
```

### Proposal Cancellation Flow

```
Proposal rejected by signer community
    ↓
Proposal remains in PENDING (below quorum)
    ↓
[OR]
    ↓
Proposal CONFIRMED, delay passes
    ↓
Quorum reaches consensus to cancel
    ↓
Proposal marked CANCELLED
    ↓
Proposal fee refunded to proposer
```

---

## Configuration Parameters

| Parameter         | Type | Recommended Value            | Purpose                |
| ----------------- | ---- | ---------------------------- | ---------------------- |
| `quorum`          | u32  | 2-3 (of total signers)       | Consensus threshold    |
| `execution_delay` | u64  | 172800 (2 days)              | Time lock duration     |
| `proposal_fee`    | i128 | 10,000,000 stroops (0.1 XLM) | Spam prevention        |
| `max_signers`     | u32  | 20                           | System scalability cap |
| `MAX_DELAY`       | u64  | 2,592,000 (30 days)          | Delay upper bound      |

---

## Future Enhancements

- **Batch Operations:** Execute multiple proposals atomically
- **Weighted Voting:** Different signer weights based on role
- **Dynamic Quorum:** Adjust quorum based on total signers
- **Proposal Templates:** Pre-approved transaction templates
- **Delegation:** Temporary signer delegation
- **Timelock Reduction:** Shortened delay for emergency proposals

---

## Testing Strategy

- **Unit Tests:** Individual contract functions
- **Integration Tests:** Cross-contract interactions
- **Security Tests:** Reentrancy, double-spending, authorization
- **Testnet Deployment:** Full system validation
- **Mainnet Staging:** Production environment simulation

---

## References

- [Stellar Documentation](https://developers.stellar.org/)
- [Soroban Smart Contracts](https://soroban.stellar.org/)
- Role-based access control patterns (generic, implementation-specific)
