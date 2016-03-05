# sipgate.io

This README documents the sipgate.io functionality. There's a [demo page](https://demo.sipgate.io), [code examples](https://github.com/sipgate/sipgate.io/tree/master/examples), and a [newsletter](http://mailing.sipgate.de/f/42695-161642/).

1. [sipgate.io vs other sipgate APIs](#sipgate.io vs other sipgate APIs)
2. [Events and XML Responses](#Pushes and XML Responses)
	* [Events](#events)
	* [Responses](#responses)

## sipgate.io vs other sipgate APIs

* __sipgate.io__ is a WebHook API.
  The sipgate server sends events to a customers HTTP server. Use sipgate.io to respond to, display or record live call events. sipgate.io does not require any other sipgate products.
* __sipgate.rest__ is a REST API.
  It processes requests initiated by a customers application. Use sipgate.rest to initiate calls and to read or change configuration data for a sipgate product. sipgate.rest also enables you to authenticate sipgate users using OAuth 2. sipgate.rest requires sipgate basic, sipgate team, simquadrat or sipgate.io.
* __sipgate.rpc__ is an XML-RPC API.
  It processes requests initiated by a customers application. The XML-RPC API is deprecated and should not be used for new projects - there are no plans to cancel it though. sipgate.rpc requires sipgate basic, sipgate team or simquadrat.

## Events and XML Responses

### Events

Event   | Configuration
--------|---------------------------------
newCall | needs to be configured in the UI
answer  | &lt;[Response](#the-xml-response)&gt; [onAnswer](#onanswer)
hangup  | &lt;[Response](#the-xml-response)&gt; [onHangup](#onhangup)
DTMF    | &lt;[Gather](#gather)&gt; [onData](#ondata)


### Responses

Response            | Answer
--------------------|--------------------------------
[Dial](#dial)       | forward the call
[Play](#play)       | play a sound file
[Gather](#gather)   | gather phone keypad presses (DTMF)
[Reject](#reject)   | reject call as busy or unavailable
[Hangup](#hangup)   | hang up

sipgate.io pushes call events on the network level to an http server over insecure (http) or secure (https) connections. For production use we highly recommend using an https secured server with either a commercial or a [Let's Encrypt](https://letsencrypt.org/) certificate. Self signed certificates are deprecated.

When a call is pushed the server can answer with an XML response to change the call routing, play back audio and gather phone key presses (DTMF).

Here's what happens during a sample call handled by sipgate.io:

* someone calls one of your phone numbers
	+ newCall event gets pushed to the default sipgate.io incoming-URL
	+ server returns XML:
		+ push to URL onHangup and onAnswer
		+ Play welcome message
		+ Dial number of agent the caller spoke to before
* you answer the phone
	+ answer event gets pushed to the onAnswer URL
* the caller hangs up
	+ hangup event gets pushed to the onHangup URL

Requirements
------------

### Usage with simquadrat

* [x] [Order a simquadrat SIM](https://www.simquadrat.de)
* [x] [Book the sipgate.io feature](https://www.simquadrat.de/feature-store/sipgate.io)
* [x] [Enter an URL for incoming/outgoing calls in the dashboard](https://www.simquadrat.de/dashboard)

### Usage with sipgate basic

* [x] [Create a free sipgate basic account](https://www.sipgate.de/go)
* [x] [Book the sipgate.io feature](https://www.sipgate.de/go/feature-store/sipgate.io)
* [x] [Enter an URL for incoming/outgoing calls in the dashboard](https://www.sipgate.de/go/dashboard)

### Usage with sipgate team

* [x] Book sipgate.io in your team account (incurs monthly cost) or [request access to our developer program (free!)](http://goo.gl/forms/8TS8kQj6kx)
* [x] [After receiving the confirmation mail enter an URL for incoming/outgoing calls in sipgate team settings](https://secure.live.sipgate.de/settings/sipgateio)


The POST request
================

When enabled, sipgate.io sends POST requests with an `application/x-www-form-urlencoded` payload for each of call involving your sipgate account. Depending on the type of request it contains the following parameters:

### New call

Parameter | Description
--------- | -----------
from      | The calling number (e.g. `"492111234567"` or `"anonymous"`)
to        | The called number (e.g. `"4915791234567"`)
direction | The direction of the call (either `"in"` or `"out"`)
event     | "newCall"
callId    | A unique alphanumeric identifier to match events to specific calls (This `callId` is the same as the `session_id`, should you [initiate the call via the sipgate XML-RPC API](http://book.sipgate.io/content/click2call.html))
user[]    | The sipgate user(s) involved. It is the name of the calling user when direction is `"out"`, or of the users receiving the call when direction is `"in"`. Group calls may be received by multiple users. In that case a `"user[]"` parameter is set for each of these users. It is always `"user[]"` (not `"user"`), even if only one user is involved.

You can simulate this POST request and test your server with a cURL command:

```sh
curl -X POST --data "from=492111234567&to=4915791234567&direction=in&event=newCall&callId=123456&user[]=Alice&user[]=Bob" http://localhost:3000
```

=======
Optional Parameter | Description
-------------------|------------
diversion          | If a call was diverted before it reached sipgate.io this contains the originally dialed number.

### Answer

If you set the ["onAnswer" attribute](#onanswer) sipgate.io will push an answer-event, when
a call is answered by the other party.

Parameter | Description
--------- | -----------
event     | "answer"
callId    | Same as in newCall-event for a specific call
user      | Name of the user who answered this call. Only incoming calls can have this parameter

You can simulate this POST request and test your server with a cURL command:

```sh
curl -X POST --data "event=answer&callId=123456&user=John+Doe" http://localhost:3000
```

### Call hangup

If you set the ["onHangup" attribute](#onhangup) sipgate.io will push a hangup-event
when the call ends.

Parameter | Description
--------- | -----------
event     | "hangup"
cause     | The cause for the hangup event (see [table](#hangup-causes) below)
callId    | Same as in newCall-event for a specific call

You can simulate this POST request and test your server with a cURL command:

```sh
curl -X POST --data "event=hangup&cause=normalClearing&callId=123456" http://localhost:3000
```

#### Hangup causes

Hangups can occur due to these causes:

Cause           | Description
--------------- | -----------
normalClearing  | One of the participants hung up after the call was established
busy            | The called party was busy
cancel          | The caller hung up before the called party picked up
noAnswer        | The called party rejected the call (e.g. through a DND setting)
congestion      | The called party could not be reached
notFound        | The called number does not exist or called party is offline


### DTMF

If you ["gather"](#gather) users' dtmf reactions, this result is pushed as an event to the url specified in the ["onData" attribute](#ondata) with the following parameters:

Parameter | Description
--------- | -----------
event     | "dtmf"
dtmf      | Digit(s) the user has entered. If no input is received, the value of dtmf will be empty.
callId    | Same as in newCall-event for a specific call

You can simulate this POST request and test your server with a cURL command:

```sh
curl -X POST --data "event=dtmf&dtmf=1&callId=123456" http://localhost:3000
```


The XML response
============
After sending the POST request sipgate.io will accept an XML response to determine what to do. Make sure to set ```application/xml``` in the ```Content-Type``` header of your response.

sipgate.io currently supports the following responses for incoming and outgoing calls:

Action            | Description
----------------- | -----------
[Dial](#dial)     | Send call to voicemail or external number
[Play](#play)     | Play a sound file
[Gather](#gather) | Collects digits that a caller enters with the telephone keypad.
[Reject](#reject) | Reject call or pretend to be busy
[Hangup](#hangup) | Hang up the call

Additional to actions, the response can specify urls which shall be called by sipgate.io on certain call-events. Specify these urls via xml-attributes in the response-tag.

Url                   | Description
--------------------- | -----------
[onAnswer](#onanswer) | Receives a POST-request as soon as someone answers the call. The response to that request is discarded.
[onHangup](#onhangup) | Receives a POST-request as soon as the call ends for whatever reason. The response to that request is discarded.

Dial
----

Redirect the call and alter your caller id ([call charges apply](https://www.simquadrat.de/tarife/mobile)).

Attribute | Possible values                                              | Default value
--------- | ------------------------------------------------------------ | -------------
callerId  | Number in [E.164](http://en.wikipedia.org/wiki/E.164) format | Account settings
anonymous | true, false                                                  | Account / phone settings

Possible targets for the dial command:

Target    | Description
--------- | -----------
Number    | Send call to an external number (has to be in [E.164](http://en.wikipedia.org/wiki/E.164) format)
Voicemail | Send call to [voicemail](https://www.simquadrat.de/feature-store/voicemail) (feature has to be booked)


**Example 1: Redirect call**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Dial>
		<Number>4915799912345</Number>
	</Dial>
</Response>
```

**Example 2: Send call to voicemail**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Dial>
		<Voicemail />
	</Dial>
</Response>
```

**Example 3: Suppress phone number**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Dial anonymous="true">
		<Number>4915799912345</Number>
	</Dial>
</Response>
```

**Example 4: Set custom caller id for outgoing call**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Dial callerId="492111234567">
		<!-- Originally dialed number, extracted from POST request -->
		<Number>4915799912345</Number>
	</Dial>
</Response>
```

Play
----

Play a given sound file.

Target    | Description
--------- | -----------
Url       | Play a sound file from a given URL


**Example 1: Play a sound file**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Play>
		<Url>http://example.com/example.wav</Url>
	</Play>
</Response>
```

**Please note:** Currently the sound file needs to be a mono 16bit PCM WAV file with a sampling rate of 8kHz. You can use conversion tools like the open source audio editor [Audacity](http://audacity.sourceforge.net/) to convert any sound file to the correct format.

Linux users might want to use ```mpg123``` to convert the file:
```sh
mpg123 --rate 8000 --mono -w output.wav input.mp3
```

Gather
----

Gather collects digits that a caller enters with the telephone keypad. The onData attribute is mandatory and takes an absolute URL as a value.

Attribute | Possible values | Default value
--------- | --------------- | -------------
type      | dtmf | dtmf
onData    | absolute URL | -
maxDigits | integer >= 1 | 1
timeout (ms) | integer >= 1 | 2000

**Nesting Rules**

The following verbs can be nested within ```<Gather>```:

* [Play](#play) The timeout starts after the sound file has finished playing. After the first digit is received the audio will stop playing.

**Example 1: DTMF with sound file and additional parameters**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Gather onData="http://localhost:3000/dtmf" maxDigits="3" timeout="10000">
		<Play>
			<Url>https://example.com/example.wav</Url>
		</Play>
	</Gather>
</Response>
```

Reject
------

Pretend to be busy or block unwanted calls.

Attribute | Possible values | Default value
--------- | --------------- | -------------
reason    | rejected, busy  | rejected


**Example 1: Reject call**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Reject />
</Response>
```

**Example 2: Reject call signaling busy**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Reject reason="busy" />
</Response>
```

Hangup
------

Hang up calls

**Example: Hang up call**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
	<Hangup />
</Response>
```

onAnswer
------------

**Example: Request notification for call being answered**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response onAnswer="http://localhost:3000/answer" />
```

onHangup
------------

**Example: Request notification for call hangup**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response onHangup="http://localhost:3000/hangup" />
```

More to come
------------
Stay tuned...



Code Examples
===============

To get you started we maintain server examples for:

* [Node.js](https://github.com/sipgate/sipgate.io/tree/master/examples/nodejs)
* [PHP](https://github.com/sipgate/sipgate.io/tree/master/examples/php)

There are also examples in:

* [C++](https://github.com/sipgate/sipgate.io/tree/master/examples/c++)
* [Clojure](https://github.com/sipgate/sipgate.io/tree/master/examples/clojure)
* [Common Lisp](https://github.com/sipgate/sipgate.io/tree/master/examples/commonlisp)
* [Go](https://github.com/sipgate/sipgate.io/tree/master/examples/go)
* [Java](https://github.com/sipgate/sipgate.io/tree/master/examples/java)
* [Perl](https://github.com/sipgate/sipgate.io/tree/master/examples/perl)
* [Prolog](https://github.com/sipgate/sipgate.io/tree/master/examples/prolog)
* [Python](https://github.com/sipgate/sipgate.io/tree/master/examples/python)
* [Ruby](https://github.com/sipgate/sipgate.io/tree/master/examples/ruby)
* [Scala](https://github.com/sipgate/sipgate.io/tree/master/examples/scala)



Troubleshooting
===============

sipgate.io Log
--------------

You can enable logging for debugging purposes from your dashboard. You will find each request and the corresponding response in the logging table.

How do I inspect network traffic?
---------------------------------

You can use ```ngrep``` to inspect the incoming requests on your side:
```sh
sudo ngrep -dany -Wbyline port 3000
```



A word about security
=====================

HTTP vs. HTTPS
--------------

We strongly encourage you to use a HTTPS server. Although we support plain HTTP connections we do not recommend pushing sensitive call details over unencrypted connections. By default sipgate.io does not accept [self-signed certificates](http://stackoverflow.com/a/10176685). If you use simquadrat or sipgate basic, you can allow self-signed certs in your dashboard. In sipgate team you cannot make an exception.

Furthermore, you can add the public key of your certificate to protect the connection against man-in-the-middle attacks. Your certificate is validated by our server.

Authentication
--------------

sipgate.io supports HTTP Basic Authentication. You can include your username and password within the URL (e.g. `https://username:password@example.com:8080`).

Emergency calls
===============

sipgate.io does not process emergency calls. Emergency calls are immediately put through to emergency services.


Help us make it better
======================

Please tell us how we can improve sipgate.io. If you have a specific feature request, found a bug or would like to add an example, please use [GitHub Issues](https://github.com/sipgate/sipgate.io/issues) or fork these docs and send a [pull request](https://github.com/sipgate/sipgate.io/pulls) with your improvements.
