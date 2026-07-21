---
tags: [futures, trading, risk, position-sizing]
---
# Advanced Futures Trading Strategies

**Chapter 1**

- Base Currency: the currency trading account is denominated in.
- Multiplier or Contract Unit ($): relationship between price and value.
- Notional Exposure per Contract ($ Base Currency) = Multiplier x Price x FX rate (Quote/Base)
- Tick Size: minimum price fluctuation.
- Tick Value = Multiplier x Tick Size.
- Spread Cost: difference between mid and bid/ask.
- Back-adjustment: seeks to create a single futures contract, *but which ignores any trading cost.*
- Total Return = Spot Return + Dividends.
- Futures Pricing Formula:
$$
F = S e^{(r-d)T}
$$
- Excess Return = Spot Return + Dividends - Interest. Where **Carry = Dividends - Interest**. Carry is an economic return from holding the position over time. Thus, when Carry is negative, the price of the Future is higher than spot.
- Conclusion: an adjusted futures price includes returns from both spot and carry.
- Note: when you buy futures, you get interest from cash posted for margin. However, this varies depending on how much leverage you take. Thus, calculations are usually done considering just the adjusted price.

	**Capital**

If we double our leverage, we double our returns, but we also double our risk. Thus, it is meaningless to talk about outright returns, rather we should discuss *risk adjusted* returns.

- Negative auto-correlation of daily returns results in annualized standard deviation being too high. 
- $Sharpe Ratio (Futures) = \frac{Mean Excess Return}{Standard Deviation}$

	**Measuring Asymmetry**

Sortino Ratio can be misleading. Thus, we use Sharpe Ratio and measure the symmetry of returns separately with the Skew. Insurance companies returns are negatively skewed.

- **Skew:** good to use monthly skew because daily and weekly can be seriously affected by a couple of extreme daily returns, and annual skew does not give enough data points for a reliable estimate. 

	**Measuring Fat Tails**

- **Kurtosis:** the counterpart of skew that measures fat tails. It is not robust and it wills swing wildly if one or two extreme days are removed or added to a data series.
- **Better Approach:** demean the returns and measure the percentiles at various points.
	- Lower Percentile Ratio = 1st Percentile / 30th Percentile
	- Upper Percentile Ratio = 99th Percentile / 70th Percentile
	For a Gaussian distribution, these are both 4.43
	- Lower Tail = Lower Percentile Ratio / 4.43
	- Upper Tail = Upper Percentile Ratio / 4.43
	These gives us an indication of how extreme the lowest and highest returns are.

**Chapter 2**

This chapter shows that the most important input for position sizing is the risk of the relevant instruments (if an instrument has higher risk, then we'd want to hold a smaller position for a given amount of capital).

- *Step 1* Measuring the risk of a single contract: annualized standard deviation of returns in dollars (Notional Value * % Standard Deviation).
	- $\sigma_{pct}$ is the annualized standard deviation
	- $\tau$ is the target risk
- *Step 2* Basic Position Scaling
$$
\sigma(Contract, Base\ Currency)=Notional\ Exposure(Base \ Currency) \cdot \sigma_{pct}
$$
$$
\sigma(Position, Base\ Currency)=\sigma(Contract, Base\ Currency) \cdot N
$$
$$
\sigma(Target, Base\ Currency)=Capital(Base \ Currency) \cdot \tau
$$
$$
\sigma(Target, Base\ Currency)=\sigma(Position, Base\ Currency)
$$
$$
N= \frac{Capital \cdot \tau}{Notional\ Exposure \cdot \sigma_{pct}} 
$$
Useful Ratios:
- $Contract\ Leverage\ Ratio=\frac{Notional\ Exposure\ per\ Contract}{Capital}$
- $Volatility\ Ratio=\frac{\tau}{\sigma_{pct}}$
- $Leverage\ Ratio=\frac{Total\ Notional\ Exposure}{Capital}$
$$
N= \frac{Volatility\ Ratio}{Contract\ Leverage\ Ratio} 
$$
$$
Leverage\ Ratio=N \cdot \frac{Notional\ Exposure\ per\ Contract}{Capital}
$$
$$
Leverage\ Ratio=Volatility\ Ratio
$$
