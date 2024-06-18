# Lines of code

https://github.com/code-423n4/2024-05-canto/blob/main/ethermint-main/app/app.go#L975-L982
https://github.com/code-423n4/2024-05-canto/blob/main/canto-main/app/app.go#L1153-L1161


## Summary
Missing zero length initialization in function GetStoreKeys()

## Vulnerability Detail
In app.go files of Canto and Ethermint app, GetStoreKeys is being used to return all the stored store keys.

However,  there is a missing zero length initialization inside the function (see line 1155). This could be problematic as the return value will include zero values depending on the length of app.keys provided.

````Solidity
File: app.go
1153: // GetStoreKeys returns all the stored store keys.
1154: func (app *Canto) GetStoreKeys() []storetypes.StoreKey {
1155: 	keys := make([]storetypes.StoreKey, len(app.keys)) //@audit there should be zero lenght initialization here
1156: 	for _, key := range app.keys {
1157: 		keys = append(keys, key)
1158: 	}
1159: 
1160: 	return keys
1161: }
````

For instance, if app.keys has 3 elements (key1, key2, key3), after the loop, the keys slice looks like this:
````
keys = []storetypes.StoreKey{"", "", "", key1, key2, key3}
````
The first three entries remain as zero-value because they were pre-allocated but never updated.
## Impact

Here are the possible impacts of missing zero initialization

Inefficient Memory Usage:
The slice keys is initially allocated with a certain length, but the append operation increases the slice's length beyond what is necessary, leading to inefficient memory usage.

Presence of Zero-Value Entries:
The resulting slice contains zero-value entries, which can lead to incorrect behavior or bugs if these entries are processed later in the code.

Unnecessary Allocations:
The initial allocation followed by appending results in unnecessary allocations, which can impact performance, especially if app.keys contains many elements.

## Proof of Concept
Let's illustrate the scenario 

1. Assume app.keys contains three keys:
````
app.keys = []storetypes.StoreKey{key1, key2, key3}
````
2. Initialization:
````
keys := make([]storetypes.StoreKey, len(app.keys))
// keys: ["", "", ""]
````
3. Appending Elements:
````
keys = append(keys, key1) // keys: ["", "", "", key1]
keys = append(keys, key2) // keys: ["", "", "", key1, key2]
keys = append(keys, key3) // keys: ["", "", "", key1, key2, key3]
````
4. Final slice: As you can see the first 3 elements represents empty values, as a result which is unnecessary.
````
keys: ["", "", "", key1, key2, key3]
````


## Tools Used
Manual review

## Recommended Mitigation Steps
Revise the function into this: put zero between storeKey and len(app,keys) to represent zero length initialization.
````Diff
func (app *Canto) GetStoreKeys() []storetypes.StoreKey {
-       keys := make([]storetypes.StoreKey, len(app.keys))
+	keys := make([]storetypes.StoreKey, 0, len(app.keys))
	for _, key := range app.keys {
		keys = append(keys, key)
	}

	return keys
}
````





## Assessed type

Other