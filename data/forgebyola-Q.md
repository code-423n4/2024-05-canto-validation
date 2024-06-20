# CANTO QA REPORT

## Summary

| No | Title |
| --- | --- |
| [L-01] | If enough epochs pass without the period being increased, the accounting can be affected |
| [L-02] | GetQueryCmd and GetTxCmd are not implemented in Ethermint::feemarket |
| [L-03] | Lack of validation of Token Pair Denom in `convertERC20` |
| [L-04] | Possible to bypass maxStandardCoinPerPool  in CoinSwap module |
| [L-05] | Unimplemented TODO in code |
| [L-06] | Several modules still use the previous method of KVStore using keys |
| [L-07] |  |

## Low Findings

### [L-01]  If enough epochs pass before the calculation for the period, the accounting of Epochs can be affected

- Lines of Code: 
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/inflation/keeper/hooks.go#L73

#### Impact
`BeforeEpochStarts` and `AfterEpochEnd` are hooks called after a successful epoch begins or ends in `inflation/keeper::hooks.go`.  When `AfterEpochEnd`, there is a calculation of the current epoch which checks if enough epochs have passed to start a new period. If enough epochs pass, the period is incremented.
```
@> if epochNumber-epochsPerPeriod*int64(period)-int64(skippedEpochs) > epochsPerPeriod {
		period++
		k.SetPeriod(ctx, period)
		period = k.GetPeriod(ctx)
		bondedRatio := k.BondedRatio(ctx)
		newProvision = types.CalculateEpochMintProvision(
			params,
			period,
			epochsPerPeriod,
			bondedRatio,
		)
		k.SetEpochMintProvision(ctx, newProvision)
	}
```
However, there's a slim possibility that multiple epochs pass without this calculation being carried out. This is due to a check earlier in the hook which doesn't throw and err but ends the hook call.
```
if epochIdentifier != epochstypes.DayEpochID {
			return
	}
```
This causes the hook to end without this calculation occuring, and the epoch still gets incremented. 
The calculation checks if the currentEpoch, period and skippedEpochs are > epochPerPeriod and the period is incremented by 1, to start a new period. 
What happens if enough epochs pass without this calculation being completed due to the early return, for example if epochPerPeriod is 10, and epochs goes from 10 --> 21 before this calculation is eventually carried out. The period now should be 3. However, due to the calculation only ever incrementing by 1, the period gets set to 2. 
This affects the entire accounting and functionality of the epochs .
Currently the epochPerPeriod is set to 365, therefore the edge case of this happening is relatively low. But if the epochPerPeriod is set to a lower value, for example 10, it is possible for this to occur. 

#### Recommendation
1. Before incrementing the period, take into account the current epoch and epochsPerPeriod, or perform a check before incrementing period. For example if epochPerPeriod is 10, and currentEpoch is 23, period should be 3. Therefore a check should be performed to confirm the correct value of period being setting it to period.

### [L-02] GetQueryCmd and GetTxCmd are not implemented in Ethermint::feemarket

- Lines of Code: 
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/x/feemarket/module.go

#### Impact
`GetQueryCmd` and `GetTxCmd` are necessary for providing Cli interfaces for interaction with the Blockchain within cosmos. It is required to be implemented within modules to ensure clients have an interface exposed for carrying out several queries and commands. However, it is not implemented within feemarket module on Ethermint. This would impact the functionality of this module.


#### Recommendation 
1. Implement `GetQueryCmd` and `GetTxCmd`within `feemarket/module.go`

### [L-03]  

- Lines of Code: 
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L85

#### Impact
Token pair Denom is not validated in `erc20::convertERC20`. This may result in the conversion of an incorrect token pair in the convertERC20 operation.

#### Recommendation
Validate the Denom of the provided token pair in `erc20::convertERC20` as it is validated within `convertCoin`