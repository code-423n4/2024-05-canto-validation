1. Incorrect module name
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/ethermint-main/go.mod#L1

The module namescurrently used in the above app.go file is:
github.com/evmos/ethermint

Since Canto is not the owner of the Github repo - `github.com/evmos/ethermint` - using the name in the module makes it impossible for others to import Canto's code.

Consider using Canto's own path for its own module name.

2. Use of deprecated NewLegacyEip712SigVerificationDecorator 
NewLegacyEip712SigVerificationDecorator function is deprecated but used in handler_options.go

`// Deprecated: NewLegacyEip712SigVerificationDecorator creates a new LegacyEip712SigVerificationDecorator
func NewLegacyEip712SigVerificationDecorator(
	ak evmtypes.AccountKeeper,
	signModeHandler authsigning.SignModeHandler,
) LegacyEip712SigVerificationDecorator {
	return LegacyEip712SigVerificationDecorator{
		ak:              ak,
		signModeHandler: signModeHandler,
	}
}`

https://github.com/evmos/ethermint/blob/fd8c2d25cf80e7d2d2a142e7b374f979f8f51981/app/ante/eip712.go#L88C1-L97C2

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/app/ante/handler_options.go#L144


3. Inconsistent error messages for invalid addresses

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L40

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L44

The ConvertCoin function handles two types of addresses: the sender address and the receiver address. However, the error messages returned for invalid addresses are inconsistent.

For the sender address, the error message is wrapped with a generic message "invalid sender address":

`_, err := sdk.AccAddressFromBech32(msg.Sender)
	if err != nil {
		return nil, errorsmod.Wrap(err, "invalid sender address")
	}`

On the other hand, for the receiver address, the error message includes the specific invalid address:

`if !common.IsHexAddress(msg.Receiver) {
		return nil, errorsmod.Wrapf(sdkerrors.ErrInvalidAddress, "invalid receiver hex address %s", msg.Receiver)
	}
`

To maintain consistency and provide more informative error messages, it is recommended to include the specific invalid address in the error message for both sender and receiver addresses.

For the sender address, the error message could be updated as follows:

`_, err := sdk.AccAddressFromBech32(msg.Sender)
if err != nil {
    return nil, errorsmod.Wrapf(err, "invalid sender address %s", msg.Sender)
}
`


4.Redundant AccAddress check
The code first check if msg.Receiver address returns an error here:

`	_, err := sdk.AccAddressFromBech32(msg.Receiver)
	if err != nil {
		return nil, errorsmod.Wrap(err, "invalid receiver address")
	}`

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L97C1-L100C3

Down the line of code, the second check is here:

`	// Error checked during msg validation
	receiver := sdk.MustAccAddressFromBech32(msg.Receiver)`

Here's the MustAccAddressFromBech32 function:

`// MustAccAddressFromBech32 calls AccAddressFromBech32 and panics on error.
func MustAccAddressFromBech32(address string) AccAddress {
	addr, err := AccAddressFromBech32(address)
	if err != nil {
		panic(err)
	}

	return addr
}`

https://github.com/cosmos/cosmos-sdk/blob/d3d6448eca2c255adcd2176a4e18d21d6c798603/types/address.go#L182C1-L189C2

MustAccAddressFromBech32 function also calls AccAddressFromBech32 and returns error if error is not nil - practically also re-checking the previous check.

Similar issue is also here:
https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L38C1-L52C1

It's suggested the code below should be removed to avoid redundancy:

`_, err := sdk.AccAddressFromBech32(msg.Receiver)
	if err != nil {
		return nil, errorsmod.Wrap(err, "invalid receiver address")
	}`


5.  Incorrect handling of non-contract accounts in token pair deletion

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L62-L70

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L121-L129

Let's consider a scenario where a token pair has been created with an address that has never been a contract account (i.e., a regular non-contract account).

In the ConvertCoin or ConvertERC20 function, when the code checks for a self-destructed contract, it will incorrectly identify the non-contract account as a self-destructed contract.

The code will proceed to delete the token pair from, even though the associated account has never been a contract account. Whereas, the code should only delete the token pair if the associated account was previously a valid ERC20 contract account and has been self-destructed.

If the account has never been a contract account, the token pair should not be deleted, or a specific error should be returned to indicate an invalid account.

It's suggested that when returning an error, the code should distinguish between self-destructed contract accounts and non-contract accounts.

6. The error returned from the k.bankKeeper.SendCoinsFromModuleToAccount function is not descriptive enough

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L251-L261


7. The following code may be refactored for better readability purpose

`if ok := balanceCoinAfter.IsEqual(expCoin); !ok {
		return nil, errorsmod.Wrapf(
			types.ErrBalanceInvariance,
			"invalid coin balance - expected: %v, actual: %v",
			expCoin, balanceCoinAfter,
		)
	}`

https://github.com/code-423n4/2024-05-canto/blob/d1d51b2293d4689f467b8b1c82bba84f8f7ea008/canto-main/x/erc20/keeper/msg_server.go#L405-L411

It could be refactored this way:

`if !balanceCoinAfter.IsEqual(expCoin) {
    return nil, errorsmod.Wrapf(
        types.ErrBalanceInvariance,
        "invalid coin balance - expected: %v, actual: %v",
        expCoin, balanceCoinAfter,
    )
}`



