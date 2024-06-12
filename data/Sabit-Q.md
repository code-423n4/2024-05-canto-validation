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

