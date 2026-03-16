# Bio Launchpad — Code Examples

Python examples for on-chain participation, claiming, and withdrawal. Assume `w3`, `account`, `ADDRESS`, `CHAIN_ID`, `ERC20_ABI`, and `LAUNCH_ABI` are defined (see main skill and `references/ABIS.md`).

## approve_and_participate

Approve the base token for the launch contract, then call `participate(bioAmt)` on the launch. Use after preflight checks (balance, allowance, `mapAddrToBios` vs `maxContribution`).

```python
def approve_and_participate(launch_contract: str, base_token: str, amount: int, max_contribution: int):
    launch_contract = Web3.to_checksum_address(launch_contract)
    base_token = Web3.to_checksum_address(base_token)

    # Preflight: check cumulative contribution against cap
    contract = w3.eth.contract(address=launch_contract, abi=LAUNCH_ABI)
    current = contract.functions.mapAddrToBios(ADDRESS).call()
    if current + amount > max_contribution:
        raise ValueError(f"Would exceed max contribution: {current} existing + {amount} new > {max_contribution}")

    # Approve
    token = w3.eth.contract(address=base_token, abi=ERC20_ABI)
    approve_tx = token.functions.approve(launch_contract, amount).build_transaction({
        "from": ADDRESS,
        "nonce": w3.eth.get_transaction_count(ADDRESS),
        "chainId": CHAIN_ID,
    })
    approve_tx["gas"] = w3.eth.estimate_gas(approve_tx)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(approve_tx).raw_transaction)
    )
    assert receipt.status == 1, "Approve transaction reverted"

    # Participate
    participate_tx = contract.functions.participate(amount).build_transaction({
        "from": ADDRESS,
        "nonce": w3.eth.get_transaction_count(ADDRESS),
        "chainId": CHAIN_ID,
    })
    participate_tx["gas"] = w3.eth.estimate_gas(participate_tx)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(participate_tx).raw_transaction)
    )
    assert receipt.status == 1, "Participate transaction reverted"
```

## claim_tokens

Fetch claim data from the API, then call `claimTokens(refundAmount, claimAmount, proof)` on the launch contract. Call `claimed(address)` first to avoid sending a claim tx if already claimed.

```python
def claim_tokens(launch_id: int):
    path = f"/api/agent-api/launches/{launch_id}/claim"
    resp = requests.get(BASE_URL + path, headers=sign_auth_headers("GET", path))
    resp.raise_for_status()
    data = resp.json()

    contract_addr = Web3.to_checksum_address(data["contractAddress"])
    contract = w3.eth.contract(address=contract_addr, abi=LAUNCH_ABI)

    # Check if already claimed
    if contract.functions.claimed(ADDRESS).call():
        raise ValueError("Wallet has already claimed for this launch")

    proof = [bytes.fromhex(p.removeprefix("0x")) for p in data["merkleProof"]]
    tx = contract.functions.claimTokens(
        int(data["refundAmount"]), int(data["claimAmount"]), proof
    ).build_transaction({
        "from": ADDRESS,
        "nonce": w3.eth.get_transaction_count(ADDRESS),
        "chainId": CHAIN_ID,
    })
    tx["gas"] = w3.eth.estimate_gas(tx)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(tx).raw_transaction)
    )
    assert receipt.status == 1, "Claim transaction reverted"
```

## withdraw_if_failed

Call `withdrawTokensIfLaunchFails()` on the launch contract when the launch failed or was cancelled. Confirm launch status with the user before calling.

```python
def withdraw_if_failed(launch_contract: str):
    contract = w3.eth.contract(
        address=Web3.to_checksum_address(launch_contract),
        abi=LAUNCH_ABI,
    )
    tx = contract.functions.withdrawTokensIfLaunchFails().build_transaction({
        "from": ADDRESS,
        "nonce": w3.eth.get_transaction_count(ADDRESS),
        "chainId": CHAIN_ID,
    })
    tx["gas"] = w3.eth.estimate_gas(tx)
    receipt = w3.eth.wait_for_transaction_receipt(
        w3.eth.send_raw_transaction(account.sign_transaction(tx).raw_transaction)
    )
    assert receipt.status == 1, "Withdraw transaction reverted"
```
