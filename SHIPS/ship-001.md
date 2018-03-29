<pre>
  SHIP: 1
  Title: URI Scheme
  Authors: Paul Garcia <paul@elba.io>
  Status: Draft
  Type: Standards Track
  Created: 2018-03-29
  Last Update: 2018-03-29
</pre>

Abstract
========

This SHIP proposes a URI scheme for making SmartHoldem payments.


Motivation
==========

The purpose of this URI scheme is to enable users to easily make payments by simply clicking links on webpages or scanning QR Codes.


Specifications
==============

=== General rules for handling (important!) ===

SmartHoldem clients MUST NOT act on URIs without getting the user's authorization.
They SHOULD require the user to manually approve each payment individually, though in some cases they MAY allow the user to automatically make this decision.

=== Operating system integration ===
Graphical SmartHoldem clients SHOULD register themselves as the handler for the "smartholdem:" URI scheme by default, if no other handler is already registered. If there is already a registered handler, they MAY prompt the user to change it once when they first run the client. Official SmartHoldem desktop client is build upon electron, which [protocol module](https://github.com/electron/electron/blob/master/docs/api/protocol.md) allow a convenient way to register and intercept a custom protocol URI scheme.

=== General Format ===

SmartHoldem URIs follow the general format for URIs as set forth in [RFC 3986](https://tools.ietf.org/html/rfc3986). The path component consists of a SmartHoldem address, and the query component provides additional payment options.

Elements of the query component may contain characters outside the valid range. These must first be encoded according to UTF-8, and then each octet of the corresponding UTF-8 sequence must be percent-encoded as described in RFC 3986.


Technical details or example of implementation

=== ABNF grammar ===
```
 smartholdemuri = "smartholdem:" smartholdemaddress [ "?" smartholdemparams ]
 smartholdemaddress = *base58
 smartholdemparams  = smartholdemparam [ "&" smartholdemparams ]
 smartholdemparam   = [ amountparam / labelparam / messageparam / otherparam ]
 amountparam    = "amount=" *digit [ "." *digit ]
 labelparam     = "label=" *qchar
 messageparam   = "message=" *qchar
 otherparam     = qchar *qchar [ "=" *qchar ]
 reqparam       = "req-" qchar *qchar [ "=" *qchar ]
 ```
 Here, "qchar" corresponds to valid characters of an RFC 3986 URI query component, excluding the "=" and "&" characters, which this BIP takes as separators.

The scheme component ("smartholdem:") is case-insensitive, and implementations must accept any combination of uppercase and lowercase letters. The rest of the URI is case-sensitive, including the query parameter keys.

=== Query Keys ===

*label: Label for that address (e.g. name of receiver)
*address: smartholdem address
*message: message that describes the transaction to the user ([[#Examples|see examples below]])
*size: amount of base smartholdem units ([[#Transfer amount/size|see below]])
*(others): optional, for future extensions

- `<smartholdemaddress>` : SmartHoldem recipient address encoded in Base58.
- `[?label=<label>]` : Recipient label string.
- `[?amount=<amount>]` : Amount in SmartHoldem Coin.
- `[?messageField=<messageField>]` : Message field string.
- `[?otherparam=<otherparam>]` : Reserved for future extensions.

#### SmartHoldem address

The SmartHoldem address must be valid and encoded in Base58.

#### Message field

Message field must be a UTF-8 encoded string. *e.g.* `messageField=Message%20for%20SmartHoldem`

#### Other param

Allow URI schemes extend by custom parameters. Parameter's value must be a UTF-8 encoded string and must not conflict with a reserved key (`amount`, `label`, `messageField`). *e.g.* `fromURL=site.com` or `refId=57458`

### Syntax

This section is non-normative and does not cover all possible syntax.

`smartholdem:<address>?[amount=<amount>][&label=<label>][&message=<message>][&otherparam=<otherparam>]`

### Examples

Just the address

` smartholdem:SUeGCt31AHwTZVcfZQwpPVL4jEUCtMMDTg`

Address with amount

` smartholdem:SUeGCt31AHwTZVcfZQwpPVL4jEUCtMMDTg?amount=55.05`

Address with label and amount

` smartholdem:SUeGCt31AHwTZVcfZQwpPVL4jEUCtMMDTg?label=Mr-Smith&amount=25.1`

Address with label, amount and message field

` smartholdem:SUeGCt31AHwTZVcfZQwpPVL4jEUCtMMDTg?label=Mr-Smith&amount=21.6&message=Message%20for%20SmartHoldem`