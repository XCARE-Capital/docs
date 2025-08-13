# Architecture Overview

XCARE uses a modular smart account design: one main account contract that owns the funds, and a set of plug‑in modules that add features (swaps, lending, permissions, recovery) without ever taking custody. Think smart wallet core + feature adapters.

## Core idea

* **Account (owner of funds)**: AccountXCareModular is the single on‑chain account that holds assets and enforces who can act.
* **Module Registry (router)**: A registry maps each function selector (the first 4 bytes of `calldata`) to a specific module.
* **Modules (capabilities)**: Small, audited contracts that implement one capability each (e.g., `swap()`, `deposit()`, `setLimits()`), executed via `delegatecall` into the account’s storage.
* **Signers & Guardians:** The active signer authorises actions; guardians can help recover access—but cannot move funds.
* **Automation (bots):** Off‑chain executors can _prepare_ calls, but the account only executes them if the active signer (or an approved policy) authorises.\


This keeps the surface area small, features composable, and custody firmly with the user.

## Main components

### **Modular Account**

* Holds balances and critical state (e.g., `activeSigner`, guardian list).
* Verifies signatures: only the active signer (or guardians for recovery) can authorise.
* Fallback router: reads the function selector from the `calldata` and forwards the call to the right module via the registry.
* Enforces security boundaries (e.g., only signer can make external value‑moving calls).

### **Constants & Registry**

* `IXCareConstants` → provides the current moduleRegistry address.
* `IModuleRegistry` → look up modules(selector) → moduleAddress.
* Upgrades to modules happen by updating the registry mapping (governed process), not by redeploying the account.

### **Modules**

* Feature‑scoped logic (e.g., Swaps, Limits, Strategy Permissions).
* Called through account’s `fallback()` using `delegatecall`, so they operate on the account’s storage.
* No module = no capability. Unknown selectors are rejected.

### Signers & Guardians

* `activeSigner`: the only party that can authorize operations.
* Guardians can confirm a recover(`newSigner`) flow. They cannot spend funds.

### Off‑chain automation (Web2.5)

* Users can log in with Web2 credentials; the platform manages session keys and UX.
* Any on‑chain action still requires the account’s on‑chain authorisation path (active signer / policy), preserving self‑custody.

## How a call flows (high level)

1. User (or bot) prepares a transaction calling the account with some calldata (e.g., `swapExactTokensForTokens(...)`).
2. The account’s `fallback()`:
   * Extracts the selector from `calldata`.
   * Asks the registry which module owns this selector.
   * Forwards to that module via `delegatecall`.
3. The module runs its logic in the account’s storage, checks policy and signer, then performs safe external calls.
4. Success or revert bubbles back to the caller. Everything is on‑chain and auditable.

### Diagram: Components & data flow

<figure><img src="../.gitbook/assets/Untitled diagram _ Mermaid Chart-2025-08-11-103959.png" alt=""><figcaption></figcaption></figure>

### Diagram: Fallback routing & selector extraction

<figure><img src="../.gitbook/assets/Untitled diagram _ Mermaid Chart-2025-08-11-104056.png" alt=""><figcaption></figcaption></figure>

## Security boundaries

* Single source of truth: Funds and critical state live only in the account.
* Allow‑list routing: If a selector isn’t registered, the account reverts.
* Signer checks: Moving value or changing critical state requires the active signer.
* Recovery, not spending: Guardians can rotate the signer; they cannot withdraw.
* Minimised modules: Each module does one thing well—easier to reason about and audit.
* `delegatecall` discipline: Modules are reviewed to avoid storage collisions and unsafe external calls.

## Example snippets

Only signer can make outward calls:

```solidity
modifier onlyActiveSigner() {
    require(msg.sender == activeSigner, "Not active signer");
    _;
}

function callExternal(address target, uint256 value, bytes calldata data)
    external
    onlyActiveSigner
    returns (bytes memory)
{
    require(target != address(0), "Invalid target");
    (bool ok, bytes memory out) = target.call{value:value}(data);
    require(ok, _revertReason(out));
    return out;
}
```

Fallback → module dispatch:

```solidity
fallback() external payable {
    bytes4 selector = msg.sig; // or extract inner selector if wrapped
    address module = ModuleRegistry(modRegistry).modules(selector);
    require(module != address(0), "Unknown module");
    (bool ok, bytes memory ret) = module.delegatecall(msg.data);
    if (!ok) assembly { revert(add(ret, 32), mload(ret)) }
}
```

Guardian‑assisted recovery:

```solidity
function recover(address newSigner, bytes calldata guardianSig) external {
    require(newSigner != address(0), "Invalid new signer");
    bytes32 msgHash = keccak256(abi.encodePacked("Recover:", newSigner));
    address recovered = ECDSA.recover(ECDSA.toEthSignedMessageHash(msgHash), guardianSig);
    require(guardians[recovered], "Not a guardian");
    emit SignerUpdated(activeSigner, newSigner);
    activeSigner = newSigner;
}
```

## Upgrade & governance model (pragmatic)

* New features ship as new modules with new selectors.
* Rollbacks are simple: point the selector back to a previous module.
* Emergency response: Unknown selectors already revert; registry can also disable a selector → module mapping.
* Audits: Modules and registry logic are auditable in isolation.
