# Arcade: Unlocking Liquidity through NFT Lending

Below finding lead to `16k$+` rewards, and 2nd position in Arcade contest on Cantina. This was a unique `medium`

## **Summary**

There is a stepwise jump vulnerability within the LoanCore affiliate fee split configuration. The `setAffiliateSplits` function, callable by an admin role, allows for modifications to the affiliate fee splits without a gradual adjustment or notification to users. This could lead to abrupt changes in protocol fees and potentially impact users relying on the existing fee structure.

## **Impact**

The stepwise jump vulnerability in the affiliate fee split configuration could have the following impacts:

- **Sudden Changes in Protocol Fees**: Admins have the authority to modify affiliate fee splits, leading to sudden changes in protocol fees. Users initiating loans may experience unexpected variations in fees without prior notification.
  
- **User Discontent**: Abrupt modifications to affiliate fee splits may result in user dissatisfaction, especially affiliates who rely on predictable earnings. Lack of transparency and communication can erode trust and confidence in the lending platform.
  
- **Operational Disruption**: Sudden changes in protocol fees could disrupt the operations of users, such as lenders and borrowers, who have established their financial strategies based on the existing fee structure. This may lead to financial losses and operational challenges.

## **Recommendations**

For addressing this issue I have the following suggestions to mitigate potential negative impacts:

- **Gradual Transition Mechanism**: Implement a gradual transition mechanism for modifying affiliate fee splits. Instead of allowing instant changes, consider a phased approach that provides users with advance notice and allows them to adapt to the new fee structure.
  
- **User Notifications**: Establish a robust notification system to inform users, especially affiliates, about upcoming changes in affiliate fee splits. Timely and clear communication is essential to manage user expectations and minimize discontent.
  
- **User-Defined Affiliate Splits**: Introduce a feature that allows users to define their preferred affiliate fee splits when initiating loans. Empowering users with control over fee distribution enhances transparency and aligns with user preferences.