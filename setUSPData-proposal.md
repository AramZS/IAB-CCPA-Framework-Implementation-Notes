# IAB CCPA Framework Modification Proposal: setUSPDNS

## Summary

The current state of CA laws around CCPA have made it clear that they expect participants in an in-scope ad request to respect browser-level signals. The current state of the IAB CCPA Framework does not designate a method for a browser or browser extension to send a signal indicating the user's preference for Do Not Sell (DNS). Without a standard methodology to support a browser-level signal there is no way that any single approach could become codified, leaving it on whoever sets the CCPA string to detect one or potentially hundreds or different signals that could come in via different formats and methods. In order to avoid fragmentation I propose that we supply an additional interface to the `__uspapi` function that could be extended by any system. 

Notably, this does not change the shape of the USP String, it simply adds another way to set that string. This does not cover systems integration tasks (like syncing the value to other systems) or validation tasks (like determining if the user is in CA) that can be left up to the implementer. 

## Interface changes

The current interface for `__uspapi` is

- `command` string 'getUSPData'
- `version` number U.S. Privacy spec version
- `callback` function `function(uspData: uspdata,
success: boolean)`

```javascript
__uspapi('getUSPData', 1, (uspData, success) => {
			if(success) {
			// do something with uspData
			} else {
			// do something else
			}
		}
);
```

This will stay the same except for a additional command strings and one additional argument. The specification would change as follows: 

The new interface for `__uspapi` would be:

- `command` enum ['getUSPData', 'setUSPDNS', 'unsetUSPDNS']
- `version` number U.S. Privacy spec version
- `callback` function `function(uspData: uspdata,
success: boolean)`
- `additionalData` null|obj `{ userInit: true|false }`

Examples: 

```javascript
__uspapi('getUSPData', 1, (uspData, success) => {
			if(success) {
			// do something with uspData
			} else {
			// do something else
			}
		}
);

__uspapi('setUSPDNS', 1, (uspData, success) => {
			if(success) {
			// do something with uspData
			} else {
			// do something else
			}
		},
		{ userInit: true }
);
```

### Version Changes

This is a non-breaking change and so I do not see the need to update the major version number, but I am interested in discussing that question if there is a wish to do so. 

### Argument Changes

The `additionalData` argument will contain relevant data important understanding a user request:

 - `userInit` is an attestation that the user's opt-out, and activation of a Do Not Sell signal, was created by a discrete user action and not set as a default.   

The `additionalData` argument will be required for the `setUSPDNS` and `unsetUSPDNS` calls to `__uspapi`

## Response Changes

There will be a change to the expected data that enters the callback, though the property should only be set in the case of `success` being set to false, that function will receive a data object in this format:

```javascript
{
 "version": 1, /* number indicating the U.S. Privacy spec version */
 "uspString": "1YNY" /* string; not applicable: “1---” */
 /* number; 1 applies, 0 doesn’t apply, -1 not set */
 "error": null | "String to alert user to reason why their set DNS request was rejected"
 }
 ```

It is expected that in the case where `success` is false the "error" property is set to a string, but in a case where the `success` value is `true`, nothing is expected on that property. 

## Examples

Four example flows: 

### 1

```javascript

__uspapi('setUSPDNS', 1, (uspData, success) => {
			if(success) {
				console.log('The user has set Do Not Sell')
			} else {
			// do something else
				window.alert('The Do Not Sell request was rejected because '+uspData.error)
			}
		},
		{ userInit: true }
);
```

With the result of `uspData` set to

```javascript
{
 "version": 1, /* number indicating the U.S. Privacy spec version */
 "uspString": "1YYY" /* string; not applicable: “1---” */
 /* number; 1 applies, 0 doesn’t apply, -1 not set */
 "error": null
 }
```

and `success` equal to true.

### 2

```javascript

__uspapi('setUSPDNS', 1, (uspData, success) => {
			if(success) {
			// do something with uspData
			} else {
			// do something else
			}
		},
		{ userInit: false }
);
```


With the result of `uspData` set to

```javascript
{
 "version": 1, /* number indicating the U.S. Privacy spec version */
 "uspString": "1YNY" /* string; not applicable: “1---” */
 /* number; 1 applies, 0 doesn’t apply, -1 not set */
 "error": "User must initiate an opt-out request"
 }
```

and `success` equal to false.

### 3

In this example the user has attempted to initiate a DNS request outside of the CA area

```javascript

__uspapi('setUSPDNS', 1, (uspData, success) => {
			if(success) {
			// do something with uspData
			} else {
			// do something else
			}
		},
		{ userInit: true }
);
```


With the result of `uspData` set to

```javascript
{
 "version": 1, /* number indicating the U.S. Privacy spec version */
 "uspString": "1YNY" /* string; not applicable: “1---” */
 /* number; 1 applies, 0 doesn’t apply, -1 not set */
 "error": "User must be in CA to set a DNS request to opt-out"
 }
```

and `success` equal to false.

### 4

```javascript

__uspapi('unsetUSPDNS', 1, (uspData, success) => {
			if(success) {
			// do something with uspData
			} else {
			// do something else
			}
		},
		{ userInit: true }
);
```


With the result of `uspData` set to

```javascript
{
 "version": 1, /* number indicating the U.S. Privacy spec version */
 "uspString": "1YNY" /* string; not applicable: “1---” */
 /* number; 1 applies, 0 doesn’t apply, -1 not set */
 "error": null
 }
```

and `success` equal to true.

## Extending to further mechanisms for USP.

These changes can be extended as-is to other USP methods. Most notably they can be extended to the [postMessage flow](https://github.com/AramZS/IAB-CCPA-Framework-Implementation-Notes#postmessage) which would more easily allow browser extensions to set DNS state. In that case the object containing `userInit` should be sent via the already provided `parameter` property. 

## Challenges 

- We rely on systems not to lie about if the signal is user initiated. Perhaps the LSPA can be extended to cover this case?