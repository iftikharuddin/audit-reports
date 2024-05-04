# Venus protocol multichain governance

Ranked top `3` in Venus contest. Cantina twitter [post](https://twitter.com/cantinaxyz/status/1786513787098468399) 


| Issue Number | Issue Name                                           | Brief Description                                                                                     | Severity |
|--------------|------------------------------------------------------|-------------------------------------------------------------------------------------------------------|----------|
| 1            | Missing Salt and Version in DOMAIN_TYPEHASH          | The `DOMAIN_TYPEHASH` lacks a salt and version string, risking hash collisions and future compatibility issues. | Low      |
| 2            | Uninformed Source LayerZero Endpoint Updates         | The `setSrcChainId` function allows critical updates without user notification, risking unexpected transaction failures. | Low      |
| 3            | Lack of Fee Validation in Execute Function           | Insufficient gas fee validation in `execute` function could lead to transaction reverts and lost gas fees. | Low      |
| 4            | No Minimum Destination Gas Enforcement               | Absence of minimum gas requirements allows potential DoS by specifying low gas values in transactions. | Low      |
| 5            | Hardcoding the `_payInZRO` Field to False            | Hardcoding `_payInZRO` to false limits flexibility in fee payment methods using ZRO tokens in the future. | Low      |
| 6            | Use of Hardcoded `zroPaymentAddress` in LZ Send      | Hardcoding `zroPaymentAddress` to address(0) in `LZ_ENDPOINT.send` could restrict flexibility and cause service disruptions. | Low      |

---

### Issue 1: Missing Salt and Version String in DOMAIN_TYPEHASH

**Summary**: 
The contract's `DOMAIN_TYPEHASH` lacks both a salt and a version string, which does not align with standard practices for EIP-712 implementations. This omission may lead to potential hash collisions and future incompatibilities.

**Impact**: 
Without a salt, the `DOMAIN_TYPEHASH` could be prone to hash collisions, potentially compromising the integrity of the EIP-712 implementation. Additionally, the absence of a version string may cause compatibility issues with future upgrades or changes to the EIP-712 standard.

**Recommendation**: 
Update the `DOMAIN_TYPEHASH` definition to include a salt and a version string to adhere to standard practices for EIP-712 implementation.

---

### Issue 2: Uninformed Source LayerZero Endpoint Updates in `setSrcChainId`

**Summary**:
The `setSrcChainId` function in the OmnichainGovernanceExecutor contract allows updating the source LayerZero endpoint ID without informing users, potentially leading to unexpected reverts for users unaware of the change.

**Impact**:
Uninformed updates to the `setSrcChainId` may result in failed transactions for users, leading to frustration and potential loss of trust in the platform's stability.

**Recommendation**:
- Pause the sending of proposals when updating the `setSrcChainId` to avoid unexpected behavior.
- Enhance user communication to include notifications about changes in `setSrcChainId`.
- Ensure that queued proposals are managed or cleared appropriately before changing the source chain ID to prevent issues during proposal processing.

---

### Issue 3: Lack of Fee Validation in OmnichainProposalSender's `execute` Function

**Summary**:
The `execute` function in OmnichainProposalSender.sol may revert if insufficient native gas is provided by the user due to a lack of fee validation.

**Impact**:
Users may lose the gas fee and fail to execute proposals if insufficient gas is supplied, potentially disrupting user operations and damaging the platform's usability.

**Recommendation**:
Implement the `estimateFees()` function within the `execute` function to ensure that users provide sufficient fees before proceeding with transactions. This improvement will help prevent transaction failures and enhance the user experience.

---

### Issue 4: No Minimum Destination Gas Enforcement

**Summary**:
The system currently lacks a mechanism to enforce a minimum gas limit for transactions initiated by users, which can lead to potential denial of service (DoS) attacks.

**Impact**:
This vulnerability allows malicious actors to intentionally specify low gas values, potentially blocking message pathways and causing transaction failures.

**Recommendation**:
Implement a `minDestGas` check on the sender side to enforce a minimum gas requirement for transactions, ensuring the successful execution of operations on the receiving end and preventing potential DoS attacks.

---

### Issue 5: Hardcoding the `_payInZRO` Field to False

**Summary**:
The `_payInZRO` field is hardcoded to false, which may limit future flexibility in fee payment options using ZRO tokens.

**Impact**:
This hardcoded value restricts the protocol's ability to adapt to future payment methods using ZRO, potentially impacting future integrations and feature enhancements.

**Recommendation**:
Modify the implementation to pass the `_payInZRO` field as an input parameter, allowing greater flexibility and accommodating future changes in fee payment methods.

---

### Issue 6: Use of Hardcoded `zroPaymentAddress` in LZ send

**Summary**:
The execute function in OmnichainProposalSender uses a hardcoded `zroPaymentAddress` (address(0)) when calling `LZ_ENDPOINT.send`, which may limit the contract's flexibility and cause potential service disruptions.

**Impact**:
Hardcoding `zroPaymentAddress` to address(0) could lead to issues if future changes require different addresses for ZRO payments, potentially causing operational disruptions.

**Recommendation**:
Allow dynamic setting of `zroPaymentAddress` to enhance contract flexibility and prepare for potential future requirements for ZRO fee payments.
