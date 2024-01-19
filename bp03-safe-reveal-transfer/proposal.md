
(initially written by domo) 

Unisat has suggested an elegant idea as an alternative to the fraction function. This is it explained in detail:

The "balance sent as fee" rule was implemented as a protective measure against spam fractionalization in ordinal-agnostic applications, a need underscored by recent challenges with OP_RETURN protocol mints. This rule ensures that balances, instead of being transferred to miners as fees, are reverted to the sender's available balance. This design not only safeguards against unintentional balance expenditure but also facilitates the subsequent solution proposed for addressing spam fractionalization issues.

A subtle modification to the block's operational rules to give priority to the "balance sent as fee" rule offers a safeguard against spam fractionalization. The most effective application of this updated rule is in the reveal transaction of a new transfer inscription process, which permits the atomic transformation of a transferable balance from one value to another. While this method can also be applied in commit transactions, its effectiveness may be reduced due to potential intervention by sophisticated actors.

For example, a user with a 100 ordi balance in its transferable form can change this to a 50 ordi balance. By using the 100 ordi transfer inscription as a fee in the reveal transaction of a new 50 ordi transfer inscription, this objective is achieved.
![telegram-cloud-photo-size-1-5161645689099365203-y](https://github.com/brc20-devs/brc20-proposals/assets/1179924/8b9abdad-50f0-4ad2-8b5f-0d74928d149f)

The advantages of this approach include its alignment with brc-20's core principle of simplicity and minimal intrusion into core functions and indexing. It also offers users complete control over their balance, unlike the fraction proposal which focuses on attack disincentivization.

The main disadvantage is the incompatibility with existing inscription tools. This drawback is somewhat lessened, as this feature is mainly necessary for advanced users.

In summary, this solution offers an elegant and non-intrusive method to tackle brc-20’s spam fractionalization issue, utilizing a mechanism introduced to protect users from accidentally spending their balances as fees and enabling atomic conversion of transferable balance sizes.

I'm intrigued by the feasibility of executing an "inverse sandwich" attack when applying this method in the commit transaction. If it proves to be a challenging task, there might be no need for additional changes.
