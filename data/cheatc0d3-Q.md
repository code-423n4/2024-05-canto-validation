# QA Report: Canto

## L-01 Excessive Use of Panics in Critical Functions Risks Application Stability

### Impact
The InitChainer function in the SimApp uses a panic to handle errors during genesis state unmarshalling, which can lead to abrupt node termination if invalid or malformed genesis data is provided. This vulnerability could prevent nodes from initializing properly during network launches or upgrades, potentially causing widespread network downtime, failed chain starts, or creating opportunities for malicious actors to disrupt the network by intentionally triggering these panics. This could result in significant delays, loss of trust in the network, and potential financial losses for users and stakeholders relying on the blockchain's continuous operation.

### Poc
The vulnerability lies in the InitChainer method of the SimApp struct, which is a critical function in Cosmos SDK applications. This method is responsible for initializing the blockchain state during genesis or when starting from a snapshot.


https://github.com/cosmos/cosmos-sdk/blob/v0.50.6/simapp/app.go#L619

```go
func (app *SimApp) InitChainer(ctx sdk.Context, req *abci.RequestInitChain) (*abci.ResponseInitChain, error) {
    var genesisState GenesisState
    if err := json.Unmarshal(req.AppStateBytes, &genesisState); err != nil {
        panic(err)
    }
    app.UpgradeKeeper.SetModuleVersionMap(ctx, app.ModuleManager.GetVersionMap())
    return app.ModuleManager.InitGenesis(ctx, app.appCodec, genesisState)
}
```

The vulnerability stems from the use of panic() to handle errors during the unmarshalling of the genesis state. This approach is problematic for several reasons:

1. Unrecoverable failure: If the unmarshalling fails for any reason the entire application will crash due to the panic. This prevents any graceful error handling or recovery attempts.

2. Lack of error propagation: The function signature includes an error return (func (...) (..., error)), but the panic prevents utilizing this for proper error handling. Errors are not propagated up the call stack in a controlled manner.

3. Limited error information: A panic only provides a stack trace, which may not give sufficient context about what went wrong during the unmarshalling process. This can make debugging and addressing issues more difficult.

4. Potential for Denial of Service: An attacker could potentially exploit this by intentionally providing malformed genesis data, causing nodes to crash repeatedly during initialization attempts.

### Mitigation
The correct approach would be to handle the error gracefully and return it:

```go
func (app *SimApp) InitChainer(ctx sdk.Context, req *abci.RequestInitChain) (*abci.ResponseInitChain, error) {
    var genesisState GenesisState
    if err := json.Unmarshal(req.AppStateBytes, &genesisState); err != nil {
        return nil, fmt.Errorf("failed to unmarshal genesis state: %w", err)
    }
    app.UpgradeKeeper.SetModuleVersionMap(ctx, app.ModuleManager.GetVersionMap())
    return app.ModuleManager.InitGenesis(ctx, app.appCodec, genesisState)
}
```

## L-02: Improper Input Validation in Configuration Parsing

### Impact
The lack of input validation for critical configuration parameters like `invCheckPeriod` and `skipUpgradeHeights` exposes the application to potential misconfigurations or malicious manipulation. This could lead to severe consequences such as degraded performance, inconsistent network behavior, or bypassed security upgrades. An attacker or even an accidental misconfiguration could set these values to extremes, potentially causing nodes to become unresponsive, consume excessive resources, or fall out of sync with the network. This could result in network-wide instability, transaction processing issues, or create windows of vulnerability, ultimately undermining the blockchain's security and reliability.

### Poc

The vulnerability in question revolves around the inadequate validation of critical configuration parameters, specifically `invCheckPeriod` and `skipUpgradeHeights`, during the parsing of application options in the NewSimApp function of the SimApp struct.

In the current implementation, the code retrieves these values from the appOpts object using the Get method and immediately casts them to the expected types without any form of validation or error checking. For invCheckPeriod, the value is cast to an unsigned integer, while for skipUpgradeHeights, it's cast to a slice of integers.

https://github.com/cosmos/cosmos-sdk/blob/v0.50.6/simapp/app.go#L323
https://github.com/cosmos/cosmos-sdk/blob/v0.50.6/simapp/app.go#L348

```go
invCheckPeriod := cast.ToUint(appOpts.Get(server.FlagInvCheckPeriod))

skipUpgradeHeights := map[int64]bool{}
for _, h := range cast.ToIntSlice(appOpts.Get(server.FlagUnsafeSkipUpgrades)) {
    skipUpgradeHeights[int64(h)] = true
}
```

This approach is fraught with potential issues. The unchecked type conversion could lead to unexpected default values being used if the provided values are not of the expected type. There's no range validation for invCheckPeriod, which could result in performance issues if set too low or effectively disable invariant checking if set too high. Similarly, skipUpgradeHeights accepts any integer values without verifying if they correspond to valid upgrade heights or if they're within a reasonable range.

Moreover, if appOpts.Get() returns nil or an unexpected type, the code will silently use default values (0 for invCheckPeriod, empty slice for skipUpgradeHeights) without any error or warning. This silent failure mode can mask configuration errors and lead to unexpected behavior.

The security implications of this vulnerability are significant. An attacker could potentially set invCheckPeriod to an extremely low value, causing the node to spend excessive resources on invariant checks, effectively creating a denial of service condition. Manipulated skipUpgradeHeights could prevent critical security upgrades from being applied, leaving the node vulnerable to known exploits.

These parameters significantly influence the node's behavior. The invCheckPeriod determines how often invariant checks are performed, which are crucial for detecting inconsistencies in the blockchain state. The skipUpgradeHeights is used to skip specific upgrade heights, a critical security feature for managing blockchain upgrades.

### Mitigation
To mitigate this vulnerability, proper validation of these critical configuration parameters is essential. This should include type checking, range validation, and explicit error handling for invalid inputs.

## L-03: Unchecked Upgrade Skip Heights in NewCanto Function

### Impact

The `NewCanto` function uses the `skipUpgradeHeights` map without validating the heights, potentially allowing invalid upgrade skip heights to be set. If exploited, this could lead to unintended skipping of crucial upgrades or execution of upgrades at incorrect block heights, potentially causing network inconsistencies or vulnerabilities.

### Poc

In the `NewCanto` function, the `skipUpgradeHeights` parameter is passed directly to the `upgradetypes.NewKeeper` function without any validation:

https://github.com/b-harvest/Canto/blob/dudong2/feat/canto-main-cosmos-sdk%40v0.50.x/app/app.go#L515

```go
app.UpgradeKeeper = upgradekeeper.NewKeeper(
		skipUpgradeHeights,
		runtime.NewKVStoreService(keys[upgradetypes.StoreKey]),
		appCodec,
		homePath,
		app.BaseApp,
		authtypes.N
```

The `skipUpgradeHeights` is a `map[int64]bool` where the keys represent block heights at which upgrades should be skipped. However, there's no check to ensure these heights are valid (e.g., positive integers, not exceeding reasonable blockchain height limits). Invalid heights could potentially:

1. Cause upgrades to be skipped at unintended heights.
2. Lead to unexpected behavior in the upgrade process.
3. Potentially allow an attacker to manipulate the upgrade schedule if they can influence this input.

The lack of validation could also mask configuration errors, making it harder to diagnose issues related to missed or incorrectly applied upgrades.

### Mitigation
Ensure that only positive heights are allowed and also include other relevant checks (e.g., maximum height limit). 


## L-04: Unconstrained Maximum Gas Limit in AnteHandler Setup

### Impact
The `setAnteHandler` function in Canto lacks an upper bound check for the `maxGasWanted` parameter, potentially allowing excessively high gas limits. If exploited on the mainnet, this could lead to block processing inefficiencies, increased susceptibility to resource exhaustion attacks, and unpredictable transaction behavior, ultimately jeopardizing network performance and stability.

### Poc

https://github.com/b-harvest/Canto/blob/dudong2/feat/canto-main-cosmos-sdk%40v0.50.x/app/app.go#L1002

```go
func (app *Canto) setAnteHandler(txConfig client.TxConfig, maxGasWanted uint64, cdc codec.BinaryCodec, simulation bool) {
    anteHandler, err := ante.NewAnteHandler(
        ante.HandlerOptions{
            // ... other options ...
            MaxTxGasWanted: maxGasWanted,
            // ... more options ...
        },
    )
    // ... error handling and setting anteHandler
}
```

While the `uint64` type prevents negative values, it does not safeguard against unreasonably high gas limits. This oversight could result in transactions with abnormally high gas limits, potentially causing block size issues or execution timeouts.

### Mitigation
Implement an upper bound check for `maxGasWanted` in the AnteHandler setup.

## L-05: Insufficient Validation of Upgrade Information in setupUpgradeHandlers

### Impact
The `setupUpgradeHandlers` function in Canto reads upgrade information from disk without comprehensive validation. If exploited in production, this could lead to the execution of unintended or malicious upgrades, potentially causing chain halts, state inconsistencies, or security vulnerabilities across the network.

### Poc
In the `setupUpgradeHandlers` function, upgrade information is read from disk and used with minimal validation:

https://github.com/b-harvest/Canto/blob/dudong2/feat/canto-main-cosmos-sdk%40v0.50.x/app/app.go#L1402

```go
upgradeInfo, err := app.UpgradeKeeper.ReadUpgradeInfoFromDisk()
if err != nil {
    panic(fmt.Errorf("failed to read upgrade info from disk: %w", err))
}

if app.UpgradeKeeper.IsSkipHeight(upgradeInfo.Height) {
    return
}

// ... (upgrade handlers and store upgrades setup)
```

This implementation only checks for read errors and whether the upgrade should be skipped based on its height. It lacks thorough validation of the `upgradeInfo` structure, potentially allowing malformed or malicious upgrade data.

An attacker who manages to manipulate the upgrade info file could potentially execute unintended upgrades, trigger upgrades at inappropriate heights, or even introduce malicious code into the network.

The current checks do not protect against scenarios where the upgrade name is invalid or maliciously crafted, the upgrade height is set to an unreasonable value, or the upgrade data is inconsistent or corrupted.

This insufficient validation could lead to execution of unintended upgrades, premature or delayed upgrades, potential panics during the upgrade process, or the introduction of vulnerabilities that could be exploited later.

The risk is significant because upgrades are critical operations in blockchain networks, often involving changes to core functionality, consensus rules, or state transitions.

### Mitigation
Implement comprehensive validation of the `upgradeInfo` structure before using it to configure upgrades. 

## L-06: Lack of Key Table Verification in Parameter Subspace Initialization

### Impact
The `initParamsKeeper` function in Canto creates parameter subspaces without verifying the associated key tables for each module. If exploited in a production environment, this could lead to improperly structured or unconstrained parameters, potentially causing unexpected behavior, parameter corruption, or security vulnerabilities that could compromise the blockchain's stability and integrity.

### Poc

https://github.com/b-harvest/Canto/blob/dudong2/feat/canto-main-cosmos-sdk%40v0.50.x/app/app.go#L1319

```go
func initParamsKeeper(appCodec codec.BinaryCodec, legacyAmino *codec.LegacyAmino, key, tkey storetypes.StoreKey) paramskeeper.Keeper {
    paramsKeeper := paramskeeper.NewKeeper(appCodec, legacyAmino, key, tkey)

    paramsKeeper.Subspace(authtypes.ModuleName)
    paramsKeeper.Subspace(banktypes.ModuleName)
    paramsKeeper.Subspace(stakingtypes.ModuleName)
    // More subspaces...

    return paramsKeeper
}
```

This implementation creates subspaces for each module without verifying the existence or correctness of their key tables. Key tables define the structure and constraints for a module's parameters, including types, default values, and validation rules.

Without key table verification, a subspace might be created with an incomplete or incorrect parameter structure. This could lead to invalid being set.

### Mitigation
Implement key table verification before creating each subspace:

1. Ensure each module provides a method to retrieve its key table.
2. Verify the key table's completeness and correctness before subspace creation.
3. Implement a fallback mechanism or error handling for modules with missing or invalid key tables.
