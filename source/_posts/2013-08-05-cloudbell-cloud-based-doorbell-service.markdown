---
layout: post
title: "Cloudbell: Cloud-based doorbell service"
date: 2013-08-05 14:40
comments: true
sharing: true
categories: Projects
alias: projects/cloudbell
---
I recently moved into a condo building with one of my college friends. The building has an intercom that allows residents to "buzz" visitors in. Usually this is done with a simple panel on the wall in the apartment. However this one is slightly more advanced. The administrator programs in a phone number to dial when a visitor rings the apartment. Whoever picks up then just has to press a button (9) and the intercom unlocks the door and ends the call. This is all well and good, if you only have one phone to pick up from. However this is the 21st century. We don't have landlines or shared phones. We all have our own cell phones or other voice provider services. What if I want the intercom to ring multiple phones? "Nope. "Can't do." said the administrator. I don't believe in "Nope. Can't be done." so I set to work researching solutions.

## Google Voice
I love Google Voice. I've used it as my primary phone provider since 2008 (back when it was super-invite-beta). One of GV's advertised features was ringing multiple phones at once. This would be perfect for what I wanted to do. Unfortunately, GV only lets you have a mobile phone associated with one account. So I would have to sacrifice my mobile phone away from my primary account and put it on the shared one. My roommate would have to do the same thing. This was not an ideal solution.

## Twilio
This is a newer service that one of my friends had played with before. I looked into what it had to offer feature-wise and quickly realized that it would do not only what I wanted, but "would-be-nice" features as well! Twilio does cost money, but not a lot of it. It's $1/mo for a phone number, $.01/min for incoming calls, and $.02/min for outgoing calls. Plenty reasonable for this project.

## System Overview
### Core Functionality
Twilio opened up interesting new features that add to the coolness of this project.

* Call routing to multiple pickup stations (users)
* Menu-based access code entry to allow pre-authorized users to enter a code and automatically open the door.

### Hosting
Twilio itself runs on their own cloud infrastructure. I use one of my Amazon S3 buckets to host the static XML files that drive the logic and flow of the project. However call menus require a dynamic web server (in this case PHP). Twilio operates some of these for free, known as Twimlets. They are simple little Twilio apps that can provide a wide variety of call functionality.

### Components
There are three major components to the Cloudbell system:

* Intercom - The unit in the entry of the building that can place outgoing calls to people, who can then hit a key and unlock the door.
* Twilio Engine - The call management system that handles call flow from the Intercom to the Pickup Stations.
* Pickup Stations - Peoples phones (mine and my roommmates) who answer calls and hit keys.


### Call Flow
A visitor walks up to the intercom and enters the apartment number (111#). The intercom has been programmed with my Twilio phone number (222-333-4444). The intercom dials the phone number.

Twilio is configured with a URL to GET when an incoming call is received. That URL is an XML file like this:
```xml
<Response>
	<Gather numDigits="4" action="http://twimlets.com/menu?Message=Enter%20access%20code%20or%20press%201%20to%20be%20connected.&Options%5B1%5D=connect.xml&Options%5B2222%5D=letthemin.xml&">
		<Say>Enter access code or press 1 to be connected.</Say>
	</Gather>
	<Redirect/>
</Response>
```
Generated at https://www.twilio.com/labs/twimlets/menu

This code will prompt the user (```<Say>```) to enter 4 digits (```numDigits="4"```) and POST the values to a URL (```action="http://"```). Lets look at the decoded URL:
```
http://twimlets.com/menu
$Message="Enter access code or press 1 to be connected."
$Options[1]=connect.xml
$Options[2222]=letthemin.xml
```
Here we can see that when the user presses 1 the application will call another bit of XML located at ```connect.xml```. Likewise when the user enters 2222 a similar bit of XML will be called at ```letthemin.xml```. These are usually fully-qualified URLs.

Lets look at what happens when they hit 1. 
```xml
<Response>
	<Say>Connecting</Say>
	<Dial action="http://twimlets.com/simulring?Dial=true&FailUrl=fail.xml&" timeout="10">
		<Number url="http://twimlets.com/whisper?Message=Someone+is+at+the+door.+Press+any+key+to+answer+and+9+to+let+them+in">111-111-1111</Number>
		<Number url="http://twimlets.com/whisper?Message=Someone+is+at+the+door.+Press+any+key+to+answer+and+9+to+let+them+in">222-222-2222</Number>
	</Dial>
</Response>
```
Here the system announces "Connecting" (```<Say>```) and dials two numbers (111-111-1111 and 222-222-2222). Whoever picks up first gets the call and a message is played: "Press any key to answer and 9 to let them in". Simulring wants the receiver to accept the call before it is connected (hence the "press any key" part). 9 is the key that tells the intercom to unlock the door. 

The ```<Dial>``` action has also been configured with a failure URL (```FailUrl```), which is a bit of code to call when no one picks up. This occurs after 10 seconds (```timeout="10"```).

fail.xml is pretty simple:
```xml
<Response>
	<Say>No one is available at this time. Please try again later.</Say>
</Response>
```
After this the call is terminated.

Now lets go back and look at the code when then user entered the correct access code (2222). In this case a new file is called located at ```letthemin.xml```.
```xml
<Response>
	<Say>Access granted</Say>
	<Play>access.wav</Play>
</Response>
```

This code says "Access granted" then plays an audio file that has been pre-generated. In this case it is the DTMF tone for 9. I generated my file at <a href="http://www.dialabc.com/sound/generate/">DialABC</a>.

By playing the tone, the intercom unlocks the door and terminates the call.

## Conclusion
This system has worked pretty well thus far. I can give my friends the access code and they can let themselves into the building without myself or my roommate doing any work. Hosting it completely in the cloud also permits a certain degree of reliability in the system.
