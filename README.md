# IAB CCPA Framework Implementation Notes

## Summary

The proposed IAB framework is relatively straightforward to implement, adding a three character string to URLs that are sent user data and making a javascript function available on all relevant pages. The IAB framework takes whatever we would define as a user opt out signal and turns it into a three-character string expressing the version of the framework, if the user was notified that they could opt out, and if the user opted out. There is also a version of the string to make it clear that the current ad/data call is not applicable to the current user. The framework then describes an interface we would need to create that any of the compliant systems could access on the page. To implement the framework, we would need to build all supporting functions; other than the shape of the interface (given the specified inputs what information we are expected to return), the framework does not supply any code.

### What does this framework do?

- Define a signal for systems to understand the user's consent state. 
- Define a way for 3rd party scripts to read the user's consent state. 
- Define a way for content in iframes to read the user’s consent state.
- Define a way for us to send the user's consent states on HTTP requests.

## Sources

- https://www.iab.com/guidelines/ccpa-framework/
- https://www.iab.com/wp-content/uploads/2019/10/IAB_CCPA_Compliance_Framework_Draft_for_Public_Comment_Oct-2019.pdf
- https://iabtechlab.com/standards/ccpa/
- https://www.iab.com/guidelines/ccpa-framework/
- https://iabtechlab.com/wp-content/uploads/2019/10/CCPA_Compliance_Framework_US_Privacy_String_IABTechLab_Draft_for_Public_Comment.pdf
- https://iabtechlab.com/wp-content/uploads/2019/10/CCPA_Compliance_Framework_US_Privacy_USER_SIGNAL_API_SPEC_IABTechLab_DRAFT_for_Public_Comment.pdf

### Finalized Specifications

- https://iabtechlab.com/wp-content/uploads/2019/11/U.S.-Privacy-String-v1.0-IAB-Tech-Lab.pdf
- https://iabtechlab.com/wp-content/uploads/2019/11/US-Privacy-USER-SIGNAL-API-SPEC-v1.0.pdf
- https://iabtechlab.com/wp-content/uploads/2019/11/OpenRTB-Extension-U.S.-Privacy-IAB-Tech-Lab.pdf

### Demo of Spec from IAB

- https://github.com/InteractiveAdvertisingBureau/CCPA-reference-code

## Technical Specification Summary: 

I have here re-authored or duplicated the relevant parts of the technical specification given by the IAB as I best understand it, by combining information from all of their technical documents. 

---

### uspString

The uspString is composed of three elements:

#### Specification

*Type*: Number  
*Default Value*: `1`  
*Current Possible Values*: [`1`]  
*Description*: Represents the version of this string specification

#### Explicit Notice

*Type*: ENUM  
*Default Value*: `N`  
*Current Possible Values*: [  
	`N` = No  
	`Y` = Yes  
	`-` = Not Applicable
]  
*Description*: Has explicit notice been provided. For instance, CCPA 1798.115(d).

#### Users' Opt-Out of Sale 

*Type*: ENUM  
*Default Value*: `N` (User data use in sale is opt-out, so by default they are opted in)  
*Current Possible Values*: [  
	`N` = No, user has not opted out of the sale.  
	`Y` = Yes, the user has opted out of their data being used for the sale.  
	`-` = Not Applicable
]  
*Description*: Has user opted-out of the sale of their data

#### Limited Service Provider Agreement

*Type*: ENUM  
*Default Value*: `Y` (If you are using the specification then you are presumably a signatory)  
*Current Possible Values*: [  
	`N` = No, the publisher has not signed the LSPA.  
	`Y` = Yes, the publisher has signed the LSPA.  
	`-` = Not Applicable
]  
*Description*: "Publisher is a signatory to the IAB Limited Service Provider Agreement (LSPA) and the publisher declares that the transaction is covered as a “Covered Opt Out Transaction” or a “Non Opt Out Transaction” as those terms are defined in the Agreement"

---

### US Privacy String


Any URL request that might send user data should have the `us_privacy` property appended to it with the uspString value set. This is an example: `http:/example.com/tracker?us_privacy=1YNY`. This includes cases where it might not apply, in which case we set the url encoded value to `1---`.

In the case where this needs to be built using ad server macros, the naming convention is `${US_PRIVACY}`.

---

### USP API

- __uspapi() must always be a function at all times, even at initialization, on the window level. 
- Secondarily, the implementation must provide a proxy for postMessage events targeted to the __uspapi interface sent from within nested iframes. 
  - Making some assumptions on how this would work, it should send an event with `getUSPData` and get the uspObject and success boolean, but the IAB framework needs more detail there. I will send in a public comment. 
- The function should be available at all times, even if the data isn't.
- You will likely want to use `function(){}.apply(null, callback)` to handle changes in the set of functions that this function will have to handle in the future. 

- `command` enum ['getUSPData', 'setUSPDNS', 'unsetUSPDNS']
- `version` number U.S. Privacy spec version
- `callback` function `function(uspData: uspdata,
success: boolean)`
- `additionalData` null|obj `{ userInit: true|false }`

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

Also assume that future iterations of the framework will need to be available in parallel with past versions of the framework in the future. 

If U.S. Privacy does not apply to this user in this context then the string in uspData object will
contain “1---”.

A value of false will be passed as the argument to the success callback when no uspData object could be returned. 

(No specification is given for what the uspData value is set to when uspData is not available because we cannot determine the users' consent state or because of script failure. Right now I think we should set it to {}, to make the failed state entirely clear.)

The callback shall be invoked only once per api call with this command.

uspData object to return:  

```javascript
{
 "version": 1, /* number indicating the U.S. Privacy spec version */
 "uspString": "1YNY" /* string; not applicable: “1---” */
 /* number; 1 applies, 0 doesn’t apply, -1 not set */
  "error": null 
  /* If success == false then this should be set to a "String to alert user to reason why their setUSPDNS request was rejected" */
 }
 ```

 It is expected that in the case where `success` is false the "error" property is set to a string, but in a case where the `success` value is `true`, nothing is expected on that property. 

 #### postMessage

The `command` and `version` object properties correspond to the parameters of the same name defined in the argument sent to the __uspapi() method. The “sent message” should also supply a unique callId property to help match the request with a response as an ad may receive multiple messages from different advertising technology. 

The postMessage function inside the iFrame should send: 

```javascript
{
	__uspapiCall:
		{
			command: "command",
			parameter: parameter,
			version: version,
			callId: uniqueId
		}
}
```

The IAB Framework does not currently define a purpose for the parameter property. 

> The returnValue object property shall be the corresponding US Privacy String object for the command used upon sending the “sent message”. The success object property shall reflect the __uspapi() success callback argument and the callId will correspond to the “sent message” unique id passed in the callId property.

The on-page function should reply to the postMessage with the following object:

```javascript
{
	__uspapiReturn:
		{
			returnValue: returnValue,
			success: boolean,
			callId: uniqueId,
			error: string
		}
}
```

`returnValue` should mirror the `uspData` object passed into the callback on-page. 

It is expected that in the case where `success` is false the "error" property is set to a string, but in a case where the `success` value is `true`, nothing is expected on that property. 

A flow might work like this: 

In an advertiser iFrame:

```javascript
topPublisherWindow.postMessage({
			__uspapiCall:
			{
				command: "getUSPData",
				parameter: null,
				version: 1,
				callId: 'xv582o'
			});
```

In a publisher page:

```javascript
  if (event.data.hasOwnProperty('__uspapiCall')) {
	__uspapi('getUSPData', 1, (uspData, success) => {
		event.source.postMessage(
			{
				__uspapiReturn:
				{
					returnValue: uspData,
					success: success,
					callId: event.data.__uspapiCall.callId
				}
			},
			event.origin);
	}
  }
  	return null;
  }
  window.addEventListener("message", receiveMessage, false);
```

In a DNS tool Extension:

```javascript
topPublisherWindow.postMessage({
			__uspapiCall:
			{
				command: "setUSPDNS",
				parameter: JSON.stringify({ userInit: true }),
				version: 1,
				callId: 'xv582g'
			});
```

In a publisher page:

```javascript
  function receiveMessage(){
	if (event.data.hasOwnProperty('__uspapiCall')) {
		__uspapi('setUSPDNS', 1, (uspData, success) => {
			event.source.postMessage(
				{
					__uspapiReturn:
					{
						returnValue: uspData,
						success: success,
						callId: event.data.__uspapiCall.callId
					}
				},
				event.origin);
		},
		JSON.parse(event.data.__uspapiCall.parameter)
	}
  	return null;
  }
  window.addEventListener("message", receiveMessage, false);
```

 #### In-App

The encoded string (ex: `1YNN` ) and any related information must be stored on  NSUserDefaults (iOS) or SharedPreferences (Android). The key/field must be `IABUSPrivacy_String` The conditions of that storage are: 

- Vendors to easily access the string information when they need to;
- The string and any related information to be persisted across app sessions;
- Pre-parsing of the string to enable all typical use-cases, with flexibility to act according to the user’s choices.

---

### OpenRTB

This OpenRTB extension defines a new attribute “us_privacy” within the BidRequest object, 

For OpenRTB v2.2+, the “us_privacy” attribute should be added into the “ext” object within the “Regs” object. For OpenRTB v2.0-2.1, the “us_privacy” attribute can be added into the “ext” object within the “User” object.

us_privacy optional string - Must follow the U.S. Privacy
string format (https://docs.google.com/document/d/1xWFNIzxvN6X8uCueXEmjfhmBrvyQqDar4b0YZfS5zpo/edit?ts=5d8c8d05#heading=h.6x5pagpxc4kn).


#### Examples:

A Digital Property has determined that U.S. Privacy applies to the transaction. The Digital Property is using version 1 of the U.S. Privacy string specification. The Digital Property has provided explicit user notice. The user has not made a choice to opt out of sale.

```javascript
{
 "Regs": {
 "ext": {
 "us_privacy": "1YNY"
 }
}
```

A Digital Property has determined that U.S. Privacy applies to the transaction. The Digital Property is using version 1 of the U.S. Privacy string specification. The Digital Property has not provided explicit user notice. The user has made a choice to opt out of sale.

```javascript
{
 "Regs": {
 "ext": {
 "us_privacy": "1NYY"
 }
}
```

A Digital Property has determined that U.S. Privacy does not apply to the transaction and is signaling this using version 1 of the U.S. Privacy string specification.

```javascript
 "Regs": {
 "ext": {
 "us_privacy": "1---"
 }
}
```

## System Requirements

- All HTTP requests to ad systems should have `?us_privacy=` and the uspString value
- The uspString value should be set to `1---` when the user is outside the enforcement zone.
- The uspString value should default to `1YNY` when the user is inside the enforcement zone. 
- After initiation, the command `__uspapi('getUSPData', 1, (uspData, success) => { console.log(uspData.version, uspData.uspString, success) } );` should:
	- Always return `1, 1---, true` outside of the enforcement zone
	- Always return `1, 1YNY, true` by default inside of the enforcement zone. 
- Inside a non-safeframe ad iframe window.top.postMessage(‘getUSPData’) should trigger a response event containing a data payload that contains the properties of uspData identical to the __uspapi function callback.
- The uspString value in all cases should change to `1YYY` when the user specifies that they have opted out. 


## AMP

- https://github.com/ampproject/amphtml/issues/24910 

## Prebid

- https://github.com/prebid/Prebid.js/blob/a30fc62119fb47740d47e1912752a4e0c73280d3/modules/consentManagementUsp.js

## Authorship of this document:

This document was produced by Aram Zucker-Scharff while examining the IAB CCPA Framework for implementation at The Washington Post
