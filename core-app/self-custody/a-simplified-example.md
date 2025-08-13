# A Simplified Example

Here’s a generic example of how this works under the hood:

```solidity
// Only the active signer can move funds
modifier onlyActiveSigner() {
    require(msg.sender == activeSigner, "Not authorized");
    _;
}

function transferFunds(address to, uint256 amount) external onlyActiveSigner {
    require(to != address(0), "Invalid recipient");
    payable(to).transfer(amount);
}
```

In this example:

* `onlyActiveSigner()` ensures no one else can call transferFunds except you.
* Even if XCARE runs automation for you, it must be signed by your account.
