Striped Obsidian Spider

high

# Borrows are not properly handled in the case of both liabilities0 and liabilities1 unable to be repaid from available assets
## Summary
After withdrawing the uniswap positions, the aggregate amount of assets( fixed assets + withdrawn uni positions + fees) is used to pay back the borrowed assets and if any one of assets has a shortfall, then the other asset is swapped into it to pay back to the lender. But if both liabilities are non-zero after repayment from the balances, nothing is swapped, whatever can be repaid is repaid and the remaining bad debt borrows is left untouched. The problem is that these left borrows will keep accruing more interest in Lender's accounting and will lead to an increased bad debt. 

## Vulnerability Detail
Borrower.sol Line 252 is meant to swap only if one asset is surplus and one is in shortfall. But the checks above that state that if liabilities0 and liabilities1 are non-zero nothing more can be done. If a a borrower account ever arrives in such a situation, then the remaining unpaid borrow amounts will be left as it is in the Lender's accounting and keep accruing more and more interest forever. 
Instead, the bad debt should be absorbed into the pool then and there and the accounting updated to make the pending borrows zero(which means shares are going to be diluted a little bit but think about the scenario if many such big positions are hanging in the borrows and accruing interest continuously, eventually the losses to lenders will be higher) and the borrower blacklisted for just this case. 

## Impact
Lenders continuously accrue losses in the form of bad debt for positions that have no left collateral and the borrower will never pay back that debt because he will lose money. Thus, if many such positions exist, the total borrows will keep growing that have no backing collateral, causing a risk of some of the lenders not being able to withdraw in the future because of this pending borrow amount.

## Code Snippet
https://github.com/sherlock-audit/2023-10-aloe/blob/b60c21af24738d517941f18f7caa8c7272f771c5/aloe-ii/core/src/Borrower.sol#L241

## Tool used

Manual Review

## Recommendation
In the liquidate function, if both liabilities0 and liabilities1 are non-zero, close the position and absorb bad debt into the lender contracts' accounting and blacklist the borrower. 