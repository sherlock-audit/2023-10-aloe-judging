Spicy Strawberry Sidewinder

high

# An attacker could manipulate prices in only one of the intervals [-2w,-w] or [-w,0] and potentially avoid detection by the metric.
## Summary
The metric calculation assumes manipulation impacts both intervals [-2w,-w] and [-w,0] equally. An attacker could focus on only one interval. 
## Vulnerability Detail
An attacker could manipulate prices in only one of the intervals [-2w,-w] or [-w,0] and potentially avoid detection by the metric.
Here is how it works:
The metric relies on taking the difference between the mean ticks in these two intervals:

       // Compute arithmetic mean tick over the interval [-2w, 0) 
       int256 meanTick0To2W = (tickCumulatives[2] - tickCumulatives[0]) / int32(UNISWAP_AVG_WINDOW * 2);

       // Compute arithmetic mean tick over the interval [-2w, -w]
       int256 meanTickWTo2W = (tickCumulatives[1] - tickCumulatives[0]) / int32(UNISWAP_AVG_WINDOW); 

       metric = uint56(SoladyMath.dist(meanTick0To2W, meanTickWTo2W));

If an attacker only manipulates prices in one interval, say [-w,0], the mean for that interval (meanTick0ToW) will be impacted but the mean for [-2w,-w] (meanTickWTo2W) will stay unchanged.
So the difference and metric may remain small and not indicate manipulation, even though prices are manipulated in [-w,0].
The impact is that the metric could fail to detect price manipulation that is occurring.

## Impact
The vulnerability of an attacker only manipulating prices in one of the intervals [-2w,-w] or [-w,0] is that the manipulation metric may not detect the manipulation.
The metric relies on taking the difference between the mean ticks in these two intervals. If an attacker only manipulates prices in one interval, the difference and hence the metric may be low and not indicate manipulation, even though prices are being manipulated.
The severity of this is high. It means the metric designed to detect manipulation could fail to do so. This allows an attacker to manipulate prices without being detected.


## Code Snippet
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Oracle.sol#L98-L100
https://github.com/sherlock-audit/2023-10-aloe/blob/main/aloe-ii/core/src/libraries/Oracle.sol#L135
## Tool used

Manual Review

## Recommendation
The contract could compute additional metrics over more time intervals. A suggestive idea:

       // Additional metrics

       int256 meanTick0ToW = (tickCumulatives[2] - tickCumulatives[1]) / UNISWAP_AVG_WINDOW; 

       int256 meanTickWToW = tickCumulatives[1] / UNISWAP_AVG_WINDOW;

       uint56 metric1 = SoladyMath.dist(meanTick0ToW, meanTickWToW); 

       uint56 metric2 = SoladyMath.dist(meanTick0To2W, meanTick0ToW);

       // Monitor all metrics over time for large deviations

This makes it harder for an attacker to manipulate only one interval without detection. The key is to compute and monitor multiple metrics across different time intervals to improve robustness of manipulation detection.
