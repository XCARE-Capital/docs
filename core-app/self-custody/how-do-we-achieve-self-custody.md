# How do we achieve Self-Custody?

XCARE wallets are smart contract accounts that belong to you.

They use a modular account system to allow different features (modules) to be added - such as swaps, lending, or advanced security - without giving anyone else access to your funds.

Here’s the key security principles:

### Only Your Signer Can Authorise Transactions

* Every wallet has an active signer - think of this as your account’s control key.
* Only the active signer can approve movements of funds.
* XCARE can suggest or automate actions (like swaps or lending), but cannot execute them without your signature.

### Guardians for Recovery

* You can set trusted guardians (friends, family, or other wallets) who can help recover your account if you lose access.
* Guardians cannot take your funds - they can only help reset your signer to a new one if you request it.

### Module-Based Permissions

* Instead of giving a single app full control, we use a module registry.
* Each module can only perform a specific type of action (e.g., swap tokens, deposit into a wallet).
* If a module is not registered, the account will reject its request.
