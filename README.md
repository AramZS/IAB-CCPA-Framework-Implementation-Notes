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

---

### US Privacy String


Any URL request that might send user data should have the `us_privacy` property appended to it with the uspString value set. This is an example: `http:/example.com/tracker?us_privacy=1YN`. This includes cases where it might not apply, in which case we set the url encoded value to `1--`.

---

### USP API

- __uspapi() must always be a function at all times, even at initialization, on the window level. 
- Secondarily, the implementation must provide a proxy for postMessage events targeted to the __uspapi interface sent from within nested iframes. 
  - Making some assumptions on how this would work, it should send an event with `getUSPData` and get the uspObject and success boolean, but the IAB framework needs more detail there. I will send in a public comment. 
- The function should be available at all times, even if the data isn't.
- You will likely want to use `function(){}.apply(null, callback)` to handle changes in the set of functions that this function will have to handle in the future. 

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

Also assume that future iterations of the framework will need to be available in parallel with past versions of the framework in the future. 

If U.S. Privacy does not apply to this user in this context then the string in uspData object will
contain “1--”.

A value of false will be passed as the argument to the success callback when no uspData object could be returned. 

(No specification is given for what the uspData value is set to when uspData is not available because we cannot determine the users' consent state or because of script failure. Right now I think we should set it to {}, to make the failed state entirely clear.)

The callback shall be invoked only once per api call with this command.

uspData object to return:  
```javascript
{
 "version": 1, /* number indicating the U.S. Privacy spec version */
 "uspString": "1YN" /* string; not applicable: “1--” */
 /* number; 1 applies, 0 doesn’t apply, -1 not set */
 }
 ```

The encoded string and any related information must be stored on *NSUserDefaults* (iOS) or *SharedPreferences* (Android). The key/field must be `IABUSPrivacy_String`

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
 "us_privacy": "1YN"
 }
}
```

A Digital Property has determined that U.S. Privacy applies to the transaction. The Digital Property is using version 1 of the U.S. Privacy string specification. The Digital Property has not provided explicit user notice. The user has made a choice to opt out of sale.

```javascript
{
 "Regs": {
 "ext": {
 "us_privacy": "1NY"
 }
}
```

A Digital Property has determined that U.S. Privacy does not apply to the transaction and is signaling this using version 1 of the U.S. Privacy string specification.

```javascript
 "Regs": {
 "ext": {
 "us_privacy": "1--"
 }
}
```

## System Requirements

- All HTTP requests to ad systems should have `?us_privacy=` and the uspString value
- The uspString value should be set to `1--` when the user is outside the enforcement zone.
- The uspString value should default to `1YN` when the user is inside the enforcement zone. 
- After initiation, the command `__uspapi('getUSPData', 1, (uspData, success) => { console.log(uspData.version, uspData.uspString, success) } );` should:
	- Always return `1, 1--, true` outside of the enforcement zone
	- Always return `1, 1YN, true` by default inside of the enforcement zone. 
- Inside a non-safeframe ad iframe window.top.postMessage(‘getUSPData’) should trigger a response event containing a data payload that contains the properties of uspData identical to the __uspapi function callback.
- The uspString value in all cases should change to `1YY` when the user specifies that they have opted out. 


## AMP

- https://github.com/ampproject/amphtml/issues/24910 

## Authorship of this document:

This document was produced by Aram Zucker-Scharff while examining the IAB CCPA Framework for implementation at The Washington Post
