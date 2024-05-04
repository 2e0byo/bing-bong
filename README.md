# Bing-Bong: a protocol for wireless doorbells

Bing-bong is a cryptographically secure[^1], transport agnostic wireless
doorbell protocol.

## Description

Doorbells need to do three things: set-up; maintenance, and ringing.  Bing-Bong
facilitates all of these whilst remaining secure by encrypting all packets
except where notified with `AES-128`.  This algorithm was chosen as the esp32
has hardware support for it.

### Components

A bing-bong network is composed of exactly one doorbell and one or more buttons.
Buttons and Doorbells MAY use any transport layer protocol to communicate (for
instance, LoRa WAN or udp) over any PHY (for instance: LoRa or WiFi). Buttons
and Doorbells MAY use any pre-arranged packet encoding. For convenience packets
are represented as JSON objects below, since nobody in the 21st century is going
to hand-code packet decoding anyway.

### Pairing

Pairing is the process whereby a button is attached to a doorbell.  It takes
place in the clear, which is a threat model appropriate to doorbells.  A future
standard MAY introduce Diffie-Hellman key exchange, but for now we've better
things to do with our lives.

The Button sends a packet to the server.  This may be sent in any way
(coordinating packet delivery belongs on the transport layer); the most
convenient is probably multicast.  This packet (in JSON notation) looks like
this:

```json
{
  "event": "pair",
  "button_id": id,
  "secret_key": key,
  "event_id": event_id
}
```

where the button id is a unique id (for instance a uuid) and the event id is a
counter, incrementing every time this request is sent.  This packet is sent in
the clear.

The doorbell SHALL respond with:

```json
{
  "event": "paired",
  "button_event_id": id,
  "button_id": id,
  "event_id": id
}
```

where the event id is a counter in the doorbell, inrementing every time this
response is sent, and the button id is that recieved above.  This packet is
`AES-128` encrypted with the key sent before. The doorbell will store this key
and use it in future communication with this button.

Upon recieving this packet the button goes into paired mode.  Should the button
not recieve an acknowledgment within 1 second, the request SHALL be repeated at
least 4 times, with an exponential backoff not totalling more than 30 seconds
accross all attempts.

All packets hereafter SHALL be encrypted and retried thusly.

### Maintenance

A doorbell MAY request status from a button at any point.  The status request
SHALL be as follows:

```json
{
  "event": "status request",
  "button_id": id,
  "button_event_id": id,
  "event_id": id
}
```

A button SHOULD respond to this request if possible (for instance if not in deep
sleep mode), and SHALL respond in this form if it responds:

```json
{
  "event": "status",
  "button_id": id,
  "button_event_id": id,
  "event_id": id,
  "runtime_s": 2343,
  "bings_bonged": 23,
  // Other data if relevant, such as
  "battery_percent": 0.45
}
```

This packet may be sent without requesting it, in which case the doorbell SHALL
process it.

A button SHALL send this information at least once a day, either by responding
to requests or by volunteering it.

### Ringing

A button SHALL send the following packet to ring:

```json
{
  "event": "bing",
  "button_id": id,
  "event_id": id
}
```

The button SHALL communicate to the user by some suitable signal that this event
has taken place.  This signal MUST NOT be identical to the signals for
successful or unsuccesful ringing.
The doorbell SHALL acknowledge this packet by sending:

```json
{
  "event": "bong",
  "button_id": id,
  "button_event_id": id,
  "event_id": id
}
```

This acknowledgment SHALL be communicated to the user by some signal distinct
from that of initialisation, which SHALL continue untill the following packet is
recieved, or one minute has elapsed, whichever is sooner:

```json
{
  "event": "bing bong",
  "button_id": id,
  "button_event_id": id,
  "event_id": id
}
```

should this packet not be recieved, the button SHALL enter the
error state.  Likewise should no response be recieved after retries to the ring
event, the button SHALL enter the error state.  This state SHALL be communicated
to the user by some signal unique to it, for instance a loud siren, glowing red
lights, and the words 'game over' on any character devices present.


[^1]: We think.
