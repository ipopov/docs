# Google Voice SMS over SMS, using shadow numbers

This document explains how to use your phone carrier's plain SMS service as a transport mechanism for Google Voice SMSes. It sounds complicated, and it does involve some upfront work, but in the long run has been much more satisfying for me than was using the native apps.

These instructions also enable you to use Google Voice with a dumbphone!

*Disclaimer: Although I am a Google employee, I'm speaking for myself here and draw on no proprietary knowledge of any internal systems, just what I've gleaned as a consumer.*

*Disclaimer 2: This is more or less a more detailed version of the [Send texts from your mobile phone](https://support.google.com/voice/answer/115116) support article.*

## Background
There are two common interfaces to Google Voice, the Google Voice Android app and the Hangouts Android app. These apps use RPCs to googleapis.com to send messages, and receive pings over [GCM](https://developers.google.com/cloud-messaging/) that notify them of new messages waiting. New messages are fetched via RPC and surfaced as "push notifications". Both protocols are flaky over poor data connections:
 - The protocols are chatty. The Google Voice one, for instance, requires five RPC roundtrips for a single message to be sent.
 - They are not pipelined. If several messages are queued up to be sent, one will not even begin sending until the previous one has finished.
 - They go over HTTPS/TCP. If the underlying connection drops in and out of existence, we have to wait several more roundtrips for the device to recognize that connectivity has returned, and then for it to reestablish connections to the API server.
 - They require a data connection in the first place.

Text (SMS) messages do not suffer from the above issues, and it is possible to use SMS as a transport layer for Google Voice messages.

## Definitions, shadow numbers
A user has:
- a *private* phone number **P**, assigned by her carrier
- a *public* phone number, **G**, her Google Voice number

Her *correspondent* has a public phone number **C**. As a property of the user's account Google Voice assigns the *(P, C)* tuple a *shadow* phone number **S**.

## Shadow numbers and phone calls
Google Voice calls go over the regular cellular network. After the first time a given number has been called, they do not require a data connection. The app implements this by assigning a "shadow" telephone number **S** to each "real" number **C** a user wishes to be able to call, routing every call through a Google Voice machine and translating phone numbers there. There is apparently a bank of thousands of phone numbers owned by Google used for this purpose.

When a user wants to call **C** "from" her Google Voice number **G**, the Android app implements this by making a phone call from **P** to the destination's assigned shadow number **S**. On the basis of the *(P, S)* tuple, Google Voice establishes a virtual phone call that appears to be from **G** to **C**.

## Shadow numbers for SMS
In the same way, an SMS message sent from private number **P** to shadow number **S** will result in an SMS being sent from gvoice number **G** to destination **C**. It is also possible to configure Google Voice to deliver messages to a user over SMS. SMSes originally sent from a correspondent **C** to the user's gvoice number **G** will be delivered to the user's phone as SMSes from shadow number **S** to private number **P**.

Thus, once a shadow number **S** has been established for a contact of hers, a user can communicate only with it, from her actual number. For convenience, she can add **S** as a private auxiliary phone number for the contact.

## Making it all work
### Choosing SMS delivery for Google Voice SMSes
1. [Disable the Hangouts integration](https://support.google.com/voice/answer/6023920?hl=en#turn-off-voice-messages) (i.e. revert to the Google Voice app)
2. In the Google Voice app, select ☰ > Settings > Sync and Notifications > Receive text messages > Via the messaging app.

### Discovering contacts' shadow numbers
- Wait for your contact to send you an SMS; until they do, you can still reach them by sending them SMS from within the Google Voice app; or,
- Look over your phone bill and try to identify phone calls you've made to them "via Google Voice". The number on your bill is that contact's shadow number.
