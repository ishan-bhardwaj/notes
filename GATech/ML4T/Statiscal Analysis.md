## Rolling Statistics

- Computing statistics (eg. mean) of rolling windows with pre-defined size, say 20 days. For each 20 days, compute the mean and plot it.
  ![Rolling Statistics](assets/rolling_statistics.png)
- Rolling std can be used to identify when the actual graph is far enough to buy or sell from the rolling mean.
- **Bollinger Bands** - When the graph is 2 std times downward or upward (called lower and upper bands) than the rolling std, then it is a buy or sell signal.
- Calculating rolling mean -

```
ax =df['SPY'].plot(title='SPY rolling mean', label='SPY')

rm_SPY = pd.rolling_mean(
  df['SPY'],        # the column for which the rolling means has to be calculated
  window=20,        # window size of 20 days
)

rm_SPY.plot(label='Rolling mean', ax=ax)    # axis object (ax) will add it to the existing plot

ax.set_xlabel('Date')
ax.set_ylabel('Price')
ax.legend(loc='upper left')

plt.show()
```

![SPY Rolling Mean](assets/spy_rolling_mean.png)

- Computing Bollinger Bands -

  - Compute rolling mean - `pd.rolling_mean(df['SPY'], window=20)`
  - Compute rolling standard deviation - `pd.rolling_std(df['SPY'], window=20)`
  - Compute upper and lower bands -

  ```
  def get_bollinger_bands(rm, rstd):
    upper_band = rm + rstd* 2
    lower_band = rm - rstd * 2
    return upper_band, lower_band
  ```

### Daily Returns

- Daily returns signifies the price changes on a particular day.
- The daily return for day `t` can be calculated as - `dr[t] = (price[t] / price[t-1]) - 1`
  ![Daily Returns](assets/daily_returns.png)
