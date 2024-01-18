
(initially written by domo) 

from Unisat has suggested an alternative idea that could provide a more elegant solution. It is explained as follows:

The "balance sent as fee" rule was implemented as a protective measure against spam fractionalization in ordinal-agnostic applications, a need underscored by recent challenges with OP_RETURN protocol mints. This rule ensures that balances, instead of being transferred to miners as fees, are reverted to the sender's available balance. This design not only safeguards against unintentional balance expenditure but also facilitates the subsequent solution proposed for addressing spam fractionalization issues.

A subtle modification to the block's operational rules to give priority to the "balance sent as fee" rule offers a safeguard against spam fractionalization. The most effective application of this updated rule is in the reveal transaction of a new transfer inscription process, which permits the atomic transformation of a transferable balance from one value to another. While this method can also be applied in commit transactions, its effectiveness may be reduced due to potential intervention by sophisticated actors.

For example, a user with a 100 ordi balance in its transferable form can change this to a 50 ordi balance. By using the 100 ordi transfer inscription as a fee in the reveal transaction of a new 50 ordi transfer inscription, this objective is achieved.
