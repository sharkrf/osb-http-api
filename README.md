# openSPOT3 (SRF-OSB) API

*Note that this repository is always updated according to the beta openSPOT3
firmware releases.*

The API uses HTTP and WebSocket interfaces.

To use the API you have to acquire a JWT ([JSON Web Token](https://jwt.io))
from the device by completing the login process (see below). The acquired JWT
stays valid for 1 day after the last valid HTTP request.

The acquired JWT can be used to open a WebSocket connection. This connection
can be used for further API communication.

## Login process

### Acquiring a JWT

- Request a token with a HTTP GET from http://openspot3.local/gettok.
- Concatenate the token and the password string.
- Hash this using SHA256 to get the digest.
- HTTP POST the token and the digest to http://openspot3.local/login which
  will respond with a JWT.

Example login process with example JSON queries:

- HTTP GET http://openspot3.local/gettok
  Response:
	```json
	{
	  "token": "1f9a8b7c"
	}
	```

- Our password is *"passw0rd"*. So we concatenate the token and the password,
  and hash it:
	```bash
	sha256("1f9a8b7cpassw0rd")
	```
	This gives us the digest *"2c476e1191ac5d38f72d9b00aca1c1a64aebe991de8c2c4806e413016844e6be"*
	This digest have to be lowercase only.

- Now we HTTP POST http://openspot3.local/login this JSON:
	```json
	{
	  "token": "1f9a8b7c",
	  "digest": "2c476e1191ac5d38f72d9b00aca1c1a64aebe991de8c2c4806e413016844e6be"
	}
	```

	The response will be:

	```json
	{
	  "hostname": "openspot3",
	  "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJkMjljODQwZSJ9.r3Oom8qVEAd1ceMMWibrMNsgu0DPgz-IG13MAzB-o5s"
	}
	```

	If the password in the digest is not matching, openSPOT3 will respond
	with a 401 Unauthorized header.

- Now we are logged in and can open a WebSocket connection with the acquired JWT.

### Opening a WebSocket connection

- Use the WebSocket URL *ws://openspot3.local/JWT* to open the connection.
  Replace JWT with the previously acquired JWT. The WebSocket protocol name is
  *openspot3*.

  JavaScript example:

  ```javascript
  var ws = new WebSocket("ws://openspot3.local/eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJkMjljODQwZSJ9.r3Oom8qVEAd1ceMMWibrMNsgu0DPgz-IG13MAzB-o5s", "openspot3");
  ```

  If the supplied JWT is not valid, openSPOT3 responds with 401 Unauthorized
  HTTP header.

## HTTP API interfaces

The HTTP API interfaces can be used for acquiring a valid JWT so a WebSocket
connection can be opened for further WebSocket API usage.

### http://openspot3.local/gettok

Doesn't take any request parameters. Returns the session token, which is
a hexadecimal representation (8 ASCII characters) of an unsigned 32-bit integer.

Response:
```json
{
  "token": "1f9a8b7c"
}
```

### http://openspot3.local/checktok

Checks the validity of the supplied JWT in the *Authorization: Bearer* field in
of the HTTP request header. *success* is 1 if it's valid. *nopass* is 1 if the
openSPOT3 has no password set. *dark_mode* is 1 if the web interface should set
the dark theme. *ap_mode* is 1 when openSPOT3 is in Wi-Fi access point mode.
*init_ran* is set to 1 if the initial setup has been completed at least once in
the past.

Response:
```json
{
  "success": 1,
  "nopass": 0,
  "dark_mode": 0,
  "hostname": "openspot3",
  "ip_address": "192.168.3.99",
  "ap_mode": 0,
  "init_ran": 1
}
```

### http://openspot3.local/login

Logs in the user if the given token and digest is valid. Returns the openSPOT3's
hostname and the JWT.

Request:
```json
{
  "token": "1f9a8b7c",
  "digest": "2c476e1191ac5d38f72d9b00aca1c1a64aebe991de8c2c4806e413016844e6be"
}
```

Response:
```json
{
  "hostname": "openspot3",
  "jwt": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiJkMjljODQwZSJ9.r3Oom8qVEAd1ceMMWibrMNsgu0DPgz-IG13MAzB-o5s"
}
```

### http://openspot3.local/auth

Starts the SharkRF Network login process. The browser will be redirected to the
SharkRF Network login page. After a successful login the openSPOT3 will be
authorized on the network so it can access various network resources.

### http://openspot3.local/check

Returns basic information about the device.

Response:

```json
{
  "product": "SRF-OSB-1.0",
  "hostname": "openspot3",
  "uid": "ab12cd34"
}
```

## WebSocket API usage

### Sending WebSocket API interface requests to the openSPOT3

All WebSocket API interface request messages are JSON objects sent as
WebSocket text messages in the following JSON structure:

```json
{
  "type": "...",
  "cgi": "...",
  "id": 0,
  "data": "..."
}
```

*type* specifies the WebSocket API request type. It can be "get" or "post".
*cgi* is the name of the WebSocket API interface which will receive the request.
*id* is a 16-bit unsigned integer. The response's *id* field will be set to
this value so request-response pairs can be identified.
*data* is the request JSON object (stringified) which will be passed to the
WebSocket API interface.

Here's an example message to the openSPOT3 which sets the active connector to
the Homebrew connector.

```json
{
  "type": "post",
  "cgi": "connector",
  "id": 65101,
  "data": "{\"connector\":2}"
}
```

Responses for WebSocket API interface requests sent to the openSPOT3 with type
"get" or "post" are sent with message type **resp** by the device. The JSON
format is the following:

```json
{
  "type": "resp",
  "cgi": "connector",
  "id": 65101,
  "resp": {"connector":2},
  "code": 200
}
```

*cgi* is the name of the WebSocket API interface which is sending the response.
*id* is the same 16-bit unsigned integer which is was set in the request.
*resp* is the response JSON object from the WebSocket API interface.
*code* is the HTTP response code from the WebSocket API interface.

Note that the response JSON object is NOT stringified.

### Receiving push messages from the openSPOT3

openSPOT3 sends push messages without a request from the API client. These are
in the following JSON format:

```json
{
  "type": "...",
  ...
}
```

Only the *type* field is fixed and all other fields are based on it's value.

#### Push message type: log

Each log line is sent as a separate message with type set to log. The JSON
format is the following:

```json
{
  "type": "log",
  "ts": 1519996555,
  "log": "nvmm: downloading verinfo"
}
```

*ts* is the UNIX timestamp for the log line.
*log* is the text for the log line.

#### Push message type: csd

Source and destination callsign database (CSD) data message is sent when a
call is started or ended and there's info about the source/destination
station available in the device's database. The JSON format is the following:

```json
{
  "type": "csd",
  "dmrids": [2161005,2161028],
  "dmrtgids": [],
  "nxdnids": [61005],
  "callsign": "HA2NON",
  "name": "Norbert",
  "city": "",
  "state": "",
  "country": "hu"
}
```

*ts* is the UNIX timestamp for the log line.
*log* is the text for the log line.

#### Push message type: calllog

Each call log entry is sent as a separate message with type set to calllog.
The JSON format is the following:

```json
{
  "type": "calllog",
  "id": "cafecafebeefbeef",
  "ts": 1519996555,
  "dst": "CQCQCQ",
  "src": "HA2NON",
  "entrytype": 4097,
  "duration": 5,
  "ber": 0,
  "loss": -1,
  "rssi": -52
}
```

*ts* is the UNIX timestamp for the call entry.
*id* is the call ID.
*dst* is the destination callsign (or ID as a string).
*src* is the source callsign (or ID as a string).
*entrytype* is a 16-bit value, the meaning of the bits:

- 0th bit (LSB): 0 - voice call, 1 - data call
- 1st bit: 0 - call from net, 1 - call from modem/openSPOT3
- 2nd bit: 0 - group call, 1 - private call
- 3rd bit: 0 - call end, 1 - call start
- 4th bit: 1 - call started with late entry
- 5th bit: 1 - call ended with timeout
- 6th bit: 0 - TS1, 1 - TS2
- 7th bit: 1 - voice announcement
- 8th bit: 1 - echo reply
- 9th bit: 1 - C4FM DN (VD) mode 1, 0 - C4FM DN (VD) mode 2
- 10th bit: 1 - C4FM voice wide or WiresX call, 0 - C4FM digital narrow mode
- 11th bit: 1 - DMR call
- 12th bit: 1 - C4FM call
- 13th bit: 1 - DSTAR call
- 14th bit: 1 - NXDN call
- 15th bit: 1 - P25 call
- 16th bit: 1 - POCSAG call
- 17th bit: 1 - Muted call

Other bits are currently unused.

*duration* is the call duration in seconds (float value).
*ber* is the BER in percent (float value). It's -1 if the value is not
available.
*loss* is packet loss in the received network stream in percent (float value).
It's -1 if the value is not available.
*rssi* is the average of the received signal strength in dBm. It's 0 if the
value is not available.

The destination callsign of DMR calls can contain 2 DMR IDs separated by a /
character if the call has been rerouted. In this case the first DMR ID is the
rerouted DMR ID, and the second is the original destination ID.

The destination callsign of C4FM calls contains the callsign and the DGID
separated by a / character.

#### Push message type: callupdate

openSPOT3 sends this message during a call. The JSON format is the following:

```json
{
  "type": "callupdate",
  "id": "cafecafebeefbeef",
  "bit_errors": 2,
  "lost": 0,
  "rssi": -67
}
```

*id* is the call ID.
*bit_errors* is the average bit errors since the last *callupdate* message.
*lost* is the lost packets in this call since the last *callupdate* message.
Note that this is 0 for calls coming from the modem.
*rssi* is the average RSSI in dBm for this call since the last *callupdate*
message. Note that this is 0 for calls coming from the network.

#### Push message type: dstarmsg

A decoded D-STAR message is sent with this type. The JSON format is the
following:

```json
{
  "type": "dstarmsg",
  "id": "cafecafebeefbeef",
  "ts": 1519996555,
  "msg": "BEER"
}
```

*id* is the call ID.
*ts* is the UNIX timestamp for the decoded D-STAR message.
*msg* contains the decoded D-STAR message.

#### Push message type: ta

A decoded DMR talker alias is sent with this type. The JSON is the following:

```json
{
  "type": "ta",
  "id": "cafecafebeefbeef",
  "ts": 1519996555,
  "ta": "HA2NON Norbert"
}
```

*id* is the call ID.
*ts* is the UNIX timestamp for the decoded D-STAR message.
*msg* contains the decoded talker alias.

openSPOT3 closes the WebSocket connection with reason code 4000 if the connection
is used from another location.

#### Push message type: state

openSPOT3 sends this message if the device's state changes. It is also
automatically sent after the WebSocket connection is opened.

The JSON is the following:

```json
{
  "type": "state",
  "state": 16
}
```

*status* can be:

- 1: Hardware error
- 2: Modem transmitting
- 3: Modem transmitting CWID
- 4: Modem can't transmit because BCLO is active
- 5: Modem receiving
- 9: AP mode on, client connected
- 11: No internet connection
- 14: Special connector is active (ex. Null, AutoCal etc.)
- 15: Connector connecting
- 16: Connector connected
- 17: Modem ignoring receive

#### Push message type: batt

openSPOT3 sends this message if the device's battery state changes. It is also
automatically sent after the WebSocket connection is opened.

The JSON is the following:

```json
{
  "type": "batt",
  "state": 1,
  "percent": 100,
  "mv": 4000,
  "temp": 30,
  "remaining_min": 0
}
```

*state* can be:

- 0: Battery ready (discharging)
- 1: Charger detect ongoing
- 2: Slow charging
- 3: Normal charging
- 4: Fast charging
- 5: Charge done
- 6: Battery charge fault
- 7: Battery charge fault, VBUS overvoltage protection
- 8: Battery charge fault, poor input
- 9: Battery charge fault, output overvoltage protection
- 10: Battery charge fault, thermal shutdown
- 11: Battery charge fault, no battery
- 12: Battery charge fault, battery overheat
- 13: Battery fault

*temp* is in degrees Celsius. *mv* is the battery voltage in millivolts.

#### Push message type: modemmode

openSPOT3 sends this message if the modem mode changes. It is also automatically
sent after the WebSocket connection is opened.

The JSON is the following:

```json
{
  "type": "modemmode",
  "mode": 1
}
```

*mode* is the active modem mode value. See the **modemmode** WebSocket API
interface for available *modem_mode* values.

#### Push message type: connector

openSPOT3 sends this message if the connector changes. It is also automatically
sent after the WebSocket connection is opened.

The JSON is the following:

```json
{
  "type": "connector",
  "id": 1
}
```

*id* is the active connector ID value. See the **connector** WebSocket API
interface for available *id* values.

#### Push message type: fwustate

openSPOT3 sends this message if there's a change in the firmware/data upgrade
status. It is also automatically sent after the WebSocket connection is opened.

The JSON is the following:

```json
{
  "type": "fwustate",
  "state": 0,
  "progress": 0,
  "retryinsecs": 0,
  "rebootat": 0,
  "dlname": "",
}
```

*fwustate* is the state of the firmware upgrade process. Possible values are:

- 0:  External flash memory not found.
- 1:  Idle.
- 2:  Checking for newer firmware versions.
- 3:  Retrying firmware version check. In this case, *retryinsecs* shows
      seconds until the retry starts.
- 4:  No newer firmware upgrade file found than the currently running version.
- 5:  Firmware upgrade is available.
- 6:  Data upgrade is available.
- 7:  Firmware and data upgrades are available.
- 8:  Preparing downloading of the upgrade.
- 9:  Downloading upgrade file indicated in *dlname*. In this case,
      *progress* shows the progress percent.
- 10:  Firmware upgrade reboot to bootloader scheduled for the UNIX timestamp
      in *rebootat*.
- 11: Firmware upgrade reboot to bootloader scheduled for the UNIX timestamp
      in *rebootat*, this timestamp has been passed, but we are waiting
      for the RX/TX delay because of a recent transmission.
- 12: Downloading authentication token.
- 13: Verifying stored data. In this case, *progress* shows the progress
      percent.
- 14: External flash memory bulk erase in progress.

#### Push message type: fwuinfo

openSPOT3 sends this message if there's new firmware upgrade version info is
available. It is also automatically sent after the WebSocket connection is opened.

The JSON is the following:

```json
{
  "type": "fwuinfo",
  "parts": [{"name": "FRW", "stage": "stable", "size": 12345, "version": 1},
    {"name": "CSD", "stage": "stable", "size": 12345, "version": "201807201842"]
}
```

Each part which has a new version available is listed in *parts*.

#### Push message type: apclientinfo

openSPOT3 sends this message if it completes an AP client info query in access
point mode.

The JSON is the following:

```json
{
  "type": "apclientinfo",
  "clients": [{"mac": "aa:bb:cc:dd:ee:ff", "ip": "1.2.3.4"},
    {"mac": "aa:bb:cc:dd:ee:ff", "ip": "N/A"}]
}
```

#### Push message type: netchk

openSPOT3 sends this message if it has completed an internet connection check.
It is also automatically sent after the WebSocket connection is opened.
The JSON format is the following:

```json
{
  "type": "netchk",
  "duration": 123
}
```

*duration* is the internet connection check duration in milliseconds. It's -1 if
the internet connection check has failed, it's 0 if no duration is available.

#### Push message type: wifirssi

openSPOT3 sends this message if it has queried the Wi-Fi RSSI dBm value and it has
been changed since the previous query. It is also automatically sent after the
WebSocket connection is opened.
The JSON format is the following:

```json
{
  "type": "wifirssi",
  "rssi": -81
}
```

#### Push message type: pwrsaving

openSPOT3 sends this message if the power saving state changes. The JSON format
is the following:

```json
{
  "type": "pwrsaving",
  "enabled": 0
}
```

*enabled* is 1 if power saving got enabled.

#### Push message type: status

openSPOT3 automatically sends the **status** WebSocket API interface's output to
all connected WebSocket API clients on every 500ms or when some fields change.
The JSON format is the following:

```json
{
  "type": "status",
  "status": { "status": 11, "lastntpsync": 1519998419, ... }
}
```

*status* field contains the output of the **status** WebSocket API interface as
a JSON object. Note that this object is NOT stringified.

#### Push message type: status-srfipconnserver

openSPOT3 sends this message instead of the *connectedto* push message when the
built-in SharkRF IP Connector Protocol server is active and a client connects
or disconnects. It is also automatically sent after the WebSocket connection
is opened. The JSON format is the following:

```json
{
  "type": "status-srfipconnserver",
  "status": { "client_connected": 0, "client_id": 0, ... }
}
```

*status* field contains the output of the **status-srfipconnserver** WebSocket
API interface as a JSON object. Note that this object is NOT stringified.

#### Push message type: rxsms

openSPOT3 sends this message if a DMR SMS message is received.

Messages are in hexadecimal UTF16BE format. Example: "BEER" = "0042004500450052"
Max. message length which can be sent is currently 256 UTF16BE characters (512
hex char pairs).

Please see the **dmrsms** WebSocket API interface's description for the valid
*calltype* and *format* values.
*frommodem* is 1 if the message is received from the modem.

The JSON format is the following:

```json
{
  "type": "rxsms",
  "dstid": 2161006,
  "srcid": 2161005,
  "calltype": 1,
  "format": 0,
  "frommodem": 1,
  "msg": "0042004500450052"
}
```

#### Push message type: txsms

openSPOT3 sends this message if a DMR SMS message is being transmitted, or
sending is over.

The JSON format is the following:

```json
{
  "status": 0
}

*status* can be:

- 0: message send has started
- 1: message transmitted successfully
- 2: message transmit failed

#### Push message type: info

openSPOT3 sends this message if the info API interface data is changed.
The JSON format is the following:

```json
{
  "type": "info",
  "info": { "hwver": ... }
}
```

*info* field contains the output of the **info** WebSocket API interface as a
JSON object. Note that this object is NOT stringified.

#### Push message type: notification

openSPOT3 sends this message if there's a message which should be displayed to
the user. The JSON format is the following:

```json
{
  "type": "notification",
  "id": 4,
  "param": "CSD"
}
```

*id* can be:

- 0: Firmware upgrade download has started.
- 1: Firmware upgrade downloaded.
- 2: Firmware upgrade download cancelled.
- 3: Firmware upgrade download failed.
- 4: Firmware upgrade download not allowed for the current SharkRF Network login.
- 5: SharkRF Network login needed for firmware upgrade download.
- 6: Data upgrade download has started. In this case param is the data type.
- 7: Data upgrade downloaded. In this case param is the data type.
- 8: Data upgrade download cancelled. In this case param is the data type.
- 9: Data upgrade download failed. In this case param is the data type.
- 10: Data upgrade download not allowed for the current SharkRF Network login.
- 11: SharkRF Network login needed for the data upgrade download.
- 12: Authentication token downloaded successfully.
- 13: Authentication token download failed.
- 14: Connector authentication failed.
- 15: Color code mismatch in received stream from modem.
- 16: DGID mismatch in received stream from modem.
- 17: RAN mismatch in received stream from modem.
- 18: DAPNET server error. In this case param is the error string.
- 19: DAPNET server error, invalid auth key.
- 20: NAC mismatch in received stream from modem.
- 21: Transceiver mode should be set to Digital Narrow (DN).
- 22: Transceiver modem should be set to Voice Wide (VW).

#### Push message type: time

openSPOT3 sends this message if the time changes. It is also automatically sent
after the WebSocket connection is opened. The JSON format is the following:

```json
{
  "type": "time",
  "now": 1532349928,
  "up": 243,
  "ntp_last_synced_at": 1532349928,
  "ntp_synced_host": "pool.ntp.org",
  "tzoffset": -7200
}
```

*now* is the UNIX timestamp of openSPOT3's current time.
*up* is the seconds elapsed since the openSPOT3 powered up.
*ntp_last_synced_at* is the timestamp of the last NTP sync event.
*ntp_synced_host* is the host of the NTP server which was last synced.
*tzoffset* is the offset in seconds for the current timezone including DST.

#### Push message type: cptimeouts

openSPOT3 sends this message if the configuration profile timeouts change, and
periodically during a call. It is also automatically sent after the WebSocket
connection is opened. The JSON format is the following:

```json
{
  "type": "cptimeouts",
  "cp": 2,
  "cp_sec": 300,
  "null_sec": 300,
  "null_pdown": 0
}
```

*cp* is the configuration profile slot number to change after the *cp_sec*
timeout.
*null_sec* is the timeout after the Null connector gets activated. If
*null_pdown* is 1 then the openSPOT3 will power down instead of activating the
Null connector.

#### Push message type: connectedto

openSPOT3 sends this message when it connects/disconnects to a server, or
the talkgroup/reflector connection state changes. If the built-in SharkRF IP
Connector Protocol Server is the active connector then the
*status-srfipconnserver* push message is sent instead of this one.
The *connectedto* message is also automatically sent after the WebSocket
connection is opened.

The JSON format is the following:

```json
{
  "type": "connectedto",
  "to": "FCS001/099",
  "server": "fcs001.xreflector.net",
  "primary": 1
}
```

*to* is set to the reflector/talkgroup ID or name where the current connector
is connected. It's empty if this information is unavailable.
*server* is the hostname or IP address of the currently connected connector
server.
*primary* is 1 if the primary server is used by the current connector, and
it's 0 if the backup server is used.

#### Push message type: crossmodecallinfo

The openSPOT3 sends this message to inform the client about where the current
call is sent to in case the current call is a cross mode call.
The JSON format is the following:

```json
{
  "type": "crossmodecallinfo",
  "id": "cafecafebeefbeef",
  "dst": "9",
  "src": "9998",
  "isprivate": 0,
  "tomodemmode": 5
}
```

*id* is the call ID.
*dst* is the destination callsign (or ID as a string). It can also be empty
in case of C4FM group calls.
*src* is the source callsign (or ID as a string).
*isprivate* is 1 if the call is sent as a private call.
*tomodemmode* indicates the modem mode the current call is converted to. See
the **modemmode** WebSocket API interface for available *tomodemmode* values.

#### Push message type: autocalstatus

The openSPOT3 sends this message periodically if the AutoCal connector is
activated.

The JSON format is the following:

```json
{
  "type": "autocalstatus",
  "state": 0,
  "progress": 59,
  "found_offset": 0,
  "restart_in_sec": 0
}
```

*state* can be:

- 0: Phase 1, searching for a transmission.
- 1: Phase 2, scanning.
- 2: Phase 3, measuring BER.
- 3: Finished.

*progress* is the progress in percent.
*found_offset* is in Hz and is set when the AutoCal process is finished.
*restart_in_sec* indicates how many seconds remaining after the AutoCal process
is finished before entering Phase 1 again.

#### Push message type: pocsagmsgqueue

The openSPOT3 sends this message after the WebSocket connection is opened.
The JSON format is the following:

```json
{
  "type": "pocsagmsgqueue",
  "queue": [
    {msg: "beer1", ric: 161005, alert: 0, ts: 1542706486, id: "b0b6228a2eba0ea6"},
    {msg: "beer2", ric: 161005, alert: 0, ts: 1542706487, id: "c1b6228a2eba0ea7"}
  ]
}
```

*queue* contains all queued POCSAG messages. *alert* is 1 if the message is an
alert (with or without *msg* content). *id* is a unique message identifier.
*ts* is the UNIX timestamp of the queue message add.

#### Push message type: pocsagqueuedmsg

The openSPOT3 sends this message when a POCSAG message has been added to the
message queue. The JSON format is the following:

```json
{
  "type": "pocsagqueuedmsg",
  "msg": {msg: "beer", ric: 161005, alert: 0, ts: 1542706745, id: "de4002de5048f86e"}
}
```

*alert* is 1 if the message is an alert (with or without *msg* content).
*id* is a unique message identifier. *ts* is the UNIX timestamp of the queue
message add.

#### Push message type: pocsagmsgsent

The openSPOT3 sends this message when a POCSAG message has been sent. The JSON
format is the following:

```json
{
  "type": "pocsagmsgsent",
  "id": "de4002de5048f86e"
}
```

*id* is a unique message identifier.

#### Push message type: pocsagmsgtimeout

The openSPOT3 sends this message when a POCSAG message has been removed from
the message queue because of a timeout. The JSON format is the following:

```json
{
  "type": "pocsagmsgtimeout",
  "id": "de4002de5048f86e"
}
```

*id* is a unique message identifier.

#### Push message type: pocsagstate

The openSPOT3 sends this message when the POCSAG state changes, or the
WebSocket connection is opened. The JSON format is the following:

```json
{
  "type": "pocsagstate",
  "state": 0,
  "sec": 0,
  "ts_nr": 15
}
```

*state* can be the following:

- 0: Idle.
- 1: Waiting for RX/TX end.
- 2: Waiting for TX delay. In this case *sec* is set to the remaining seconds.
- 3: Waiting for timeslot. In this case *sec* is set to the remaining seconds,
     and *ts_nr* is set to the (decimal) timeslot number which the OS2 waits.
- 4: Transmitting POCSAG messages.

#### Push message type: dapnetbgstate

The openSPOT3 sends this message when the DAPNET connector background state
changes, or the WebSocket connection is opened.
The JSON format is the following:

```json
{
  "type": "dapnetbgstate",
  "state": 0
}
```

*state* can be the following:

- 0: Uninitialized.
- 1: Initializing.
- 2: Connecting.
- 3: Connected.

#### Push message type: modemfreq

The openSPOT3 sends this message when the modem frequency changes.
The JSON format is the following:

```json
{
  "type": "modemfreq",
  "data": {rx_frequency: 433900000, dmr_rx_offset: 0, ...}
}
```

*data* field contains the output of the **modemfreq** WebSocket API interface
as a JSON object. Note that this object is NOT stringified.

#### Push message type: aprsmsgqueue

The openSPOT3 sends this message after the WebSocket connection is opened.
The JSON format is the following:

```json
{
  "type": "aprsmsgqueue",
  "queue": [
    {"is_outbound":0,"is_unconfirmed":0,"callsign":"HG1MA","msg":"beer1","id":"123","ts":1549976403},
    {"is_outbound":1,"is_unconfirmed":0,"callsign":"HG1MA","msg":"beer2","id":"124","ts":1549976405}
  ]
}
```

*queue* contains all messages in the APRS queue. *ts* is the UNIX timestamp of
the queue message add.

#### Push message type: aprsmsg

The openSPOT3 sends this message when a message has been added to the queue.
The JSON format is the following:

```json
{
  "type": "aprsmsg",
  "msg": {"is_outbound":0,"is_unconfirmed":0,"callsign":"HG1MA","msg":"beer1","id":"123","ts":1549976403}
}
```

*ts* is the UNIX timestamp of the queue message add.

#### Push message type: aprstxmsgwaitack

The openSPOT3 sends this message when an outbound message in the queue has been
sent and it is waiting for an acknowledge from the recipient, and also after
the WebSocket connection is opened.
The JSON format is the following:

```json
{
  "type": "aprstxmsgwaitack",
  "msg": {"is_outbound":1,"is_unconfirmed":0,"callsign":"HG1MA","msg":"beer1","id":"123","ts":1549976403},
  "remaining_sec": 15
}
```

*remaining_sec* is the seconds remaining until the device retries sending.

#### Push message type: aprstxmsggotack

The openSPOT3 sends this message when an outbound message has been acknowledged
by the recipient.
The JSON format is the following:

```json
{
  "type": "aprstxmsggotack",
  "id": "123"
}
```

#### Push message type: aprstxmsgtimeout

The openSPOT3 sends this message when an outbound message send has been timed
out.
The JSON format is the following:

```json
{
  "type": "aprstxmsgtimeout",
  "id": "123"
}
```

#### Push message type: aprstxmsgcancel

The openSPOT3 sends this message when an outbound message send has been
cancelled.
The JSON format is the following:

```json
{
  "type": "aprstxmsgcancel",
  "id": "123"
}
```

#### Push message type: rtc

openSPOT3 sends this message if the RTC settings are changed. It is also
automatically sent after the WebSocket connection is opened. The JSON format
is the following:

```json
{
  "type": "rtc",
  "wakeupat": 100,
  "powerdownat": 200
}
```

*wakeupat* and *powerdownat* are seconds after midnight (local time).

## WebSocket API interfaces

### logout

Logs out the user.

Response:
```json
{}
```

### reboot

Triggers a reboot.

If *reset_config* is set to 1, it resets the current configuration profile to
defaults after rebooting.
If *reset_all_configs* is set to 1, it resets all configuration profiles to
defaults after rebooting.
If *bootloader* is 1, it reboots the unit to the bootloader.

Request (optional):
```json
{
  "reset_config": 0,
  "reset_all_configs": 1,
  "bootloader": 0
}
```
Response:
```json
{}
```

### status

Returns openSPOT3's current status. Sending a get request to this interface is
not necessarily needed as openSPOT3 sends the output of this interface as
WebSocket API message type **status** automatically every 500ms or when some
fields change.

*rssi_values_dbm* contains RSSI values since the last **status** report.

*dejitter_buf_pkts* contain dejitter buffer packet count values since the
last **status** report.

*ber_values* contain the count of erroneous bits since the last **status**
report. The second number in each number pair is used for internal debugging
purposes.

*invalid_seqnums* contains the count of received invalid sequence number
errors since the last **status** report.

The traffic counters are monotonically increasing 32 bit values. *rx_bytes* and
*tx_bytes* are the total network traffic, *gw_rx_bytes* and *gw_tx_bytes* are
the traffic from/to the default gateway (usually this is the internet traffic)
and *cn_rx_bytes* and *cn_tx_bytes* are the connector traffic.

Response:
```json
{
  "rssi_values_dbm": [-60,-62,-65],
  "dejitter_buf_pkts": [0, 1, 2],
  "ber_values": [[0,0], [1,0], [2,0]],
  "invalid_seqnums": 5,
  "rx_bytes": 1442100,
  "tx_bytes": 1442100,
  "gw_rx_bytes": 144210,
  "gw_tx_bytes": 144210,
  "cn_rx_bytes": 14421,
  "cn_tx_bytes": 14421
}
```

### dmrsms

This interface handles DMR SMS sending and SMS settings when the modem is in
DMR mode. Otherwise it'll return 400 Bad Request.

Call types: 0 - private, 1 - group.
Format IDs: 0 - ETSI, 1 - UDP, 2 - UDP/Chinese. See user manual for more info.
Setting a new *send_srcid* in the query overwrites the *default_srcid*.
If *send_to_modem* is 0, the SMS will be sent to the currently active connector.
If *intercept_net_msgs* is 1, then SMS messages coming from the network to
the *default_srcid* will be processed.
If *send_intercepted_to_pocsag_ric* is not 0, received intercepted messages to
*default_srcid* will be queued as POCSAG messages to RIC set by this field.
If *only_save* is 1, the SMS will not get sent, only the *send_srcid* and
*intercept_net* settings will be stored.

Messages are in hexadecimal UTF16BE format. Example: "BEER" = "0042004500450052"
Max. message length which can be sent is currently 256 UTF16BE characters (512
hex char pairs).

Also see the **txsms** WebSocket API message type for the message send status.

Request (optional):
```json
{
  "only_save": 0,
  "intercept_net_msgs": 0,
  "send_intercepted_to_pocsag_ric": 0,
  "send_dstid": 2161005,
  "send_calltype": 0,
  "send_srcid": 9998,
  "send_format": 0,
  "send_tdma_channel": 0,
  "send_to_modem": 0,
  "send_msg": "0042004500450052"
}
```

Response:
```json
{
  "default_srcid": 9998,
  "intercept_net": 0,
  "send_intercepted_to_pocsag_ric": 0
}
```

### status-srfipconnserver

Returns the SharkRF IP Connector Server's current status, if it's the active
connector. Otherwise it'll return 400 Bad Request.

*client_connected* is 1 if a client is connected.

Response:
```json
{
  "client_connected": 0,
  "client_id": 1234,
  "client_callsign": ""
}
```

### connector

openSPOT3's active connector query (get)/change (post).

Valid connector IDs:

- 0: No connector set.
- 1: DMRplus
- 2: Homebrew
- 3: DCS/XLX
- 4: FCS
- 5: SharkRF IP Connector Client
- 6: SharkRF IP Connector Server
- 7: AutoCal
- 8: REF/XRF
- 9: YSFReflector
- 10: DAPNET
- 11: NXDNReflector
- 12: APRS
- 13: P25Reflector

Request (optional):
```json
{
  "connector": 0
}
```
Response:
```json
{
  "connector": 0
}
```

### connectorsettings

Connector other settings query (get)/change (post).

*rxtimeout_sec* is the change to null connector timeout.
*rxtimeout_mode* can be 0 (modem) or 1 (modem or network).

Request (optional):
```json
{
  "rxtimeout_sec": 0,
  "rxtimeout_mode": 0
}
```
Response:
```json
{
  "rxtimeout_sec": 0,
  "rxtimeout_mode": 0
}
```

### nullsettings

Null connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0
}
```

### dmrplussettings

DMRplus connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

*server_id* is the IEEE CRC32 of the server's name from the server list.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "server_id": 123,
  "port": 8880,
  "dmr_id": "",
  "reflector_id": 0,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10,
  "conn_retry_interval_sec": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 8880,
  "dmr_id": "",
  "reflector_id": 0,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10,
  "conn_retry_interval_sec": 1
}
```

### homebrewsettings

Homebrew connector settings query (get)/change (post).

Valid *autocon_calltype*, *crossmode_dstcalltype* and *reroute_calltype* values:
0 - group call, 1 - private call.

For more information about the MMDVM options field, see
[this](https://github.com/g4klx/MMDVMHost/blob/master/DMRplus_startup_options.md)
description.

See the **modemmode** interface for available *modem_mode* values.

*is_bm_srv* should be set to 1 if the server is a BrandMeister server.
*server_name* should be set to the currently selected server's name.

Fields starting with *b_* are for the backup server.
*b_toggle_timeout_sec* is the timeout in seconds to toggle between the backup
and primary servers.

*server_name* is the server name from the server list.
*server_id* is the BrandMeister server's ID. If the server is not a
BrandMeister server, then it is the IEEE CRC32 of the server's name from the
server list.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "server_name": "",
  "server_id": 123,
  "is_bm_srv": 0,
  "server_name": "",
  "port": 62030,
  "mmdvm_mode": 0,
  "mmdvm_options": "",
  "callsign": "",
  "password": "",
  "b_server_host": "",
  "b_is_bm_srv": 0,
  "b_server_name": "",
  "b_port": 62030,
  "b_password": "",
  "b_toggle_timeout_sec": 60,
  "repeater_id": 901234,
  "url": "https://www.sharkrf.com/",
  "autocon_id": 4771,
  "autocon_calltype": 0,
  "autocon_tdma_channel": 0,
  "autocon_interval_sec": 500,
  "autocon_discon": 0,
  "dmo_tdma_channel": 0,
  "crossmode_dstid": 9,
  "crossmode_dstcalltype": 0,
  "reroute_id": 9990,
  "reroute_calltype": 1,
  "dynreroute_disabled": 0,
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 62030,
  "mmdvm_mode": 0,
  "mmdvm_options": "",
  "callsign": "",
  "password": "",
  "b_server_host": "",
  "b_port": 62030,
  "b_password": "",
  "b_toggle_timeout_sec": 60,
  "repeater_id": 901234,
  "url": "https://www.sharkrf.com/",
  "autocon_id": 4771,
  "autocon_tdma_channel": 0,
  "autocon_interval_sec": 500,
  "autocon_discon": 0,
  "dmo_tdma_channel": 0,
  "crossmode_dstid": 9,
  "crossmode_dstcalltype": 0,
  "reroute_id": 9990,
  "reroute_calltype": 1,
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1
}
```

### dcsxlxsettings

DCS/XLX connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 12345,
  "ccs_port": 12345,
  "callsign": "",
  "local_module": "A",
  "reflector": "DCS025",
  "remote_module": "Z",
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1,
  "auto_set_urcall_to_net": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 12345,
  "ccs_port": 12345,
  "callsign": "",
  "local_module": "A",
  "reflector": "DCS025",
  "remote_module": "Z",
  "rx_timeout_sec": 1,
  "conn_retry_interval_sec": 1,
  "auto_set_urcall_to_net": 1
}
```

### refxrfsettings

REF/XRF connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 12345,
  "ccs_port": 12345,
  "callsign": "",
  "local_module": "D",
  "reflector": "REF001",
  "remote_module": "C",
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1,
  "gw_mode": 0,
  "auto_set_urcall_to_net": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 12345,
  "ccs_port": 12345,
  "callsign": "",
  "local_module": "D",
  "reflector": "REF001",
  "remote_module": "C",
  "allow_g": 1,
  "rx_timeout_sec": 1,
  "conn_retry_interval_sec": 1,
  "gw_mode": 0,
  "auto_set_urcall_to_net": 1
}
```

### fcssettings

FCS connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "fcs001.xreflector.net",
  "port": 12345,
  "callsign": "",
  "ccs7_id": 2161005,
  "reflector": "FCS001",
  "room_number": 25,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "fcs001.xreflector.net",
  "port": 12345,
  "callsign": "",
  "ccs7_id": 2161005,
  "reflector": "FCS001",
  "room_number": 25,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10,
  "conn_retry_interval_sec": 1
}
```

### ysfrefsettings

YSFReflector connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 42000,
  "callsign": "",
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 42000,
  "callsign": "",
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10,
  "conn_retry_interval_sec": 1
}
```

### nxdnrefsettings

NXDNReflector connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 41400,
  "callsign": "",
  "tg_id": 123,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 41400,
  "callsign": "",
  "tg_id": 123,
  "default_cross_mode_dstid": 9,
  "always_use_cross_mode_dstid": 1,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10,
  "conn_retry_interval_sec": 1
}
```

*default_cross_mode_dstid* and *always_use_cross_mode_dstid* are the same
fields as in **nxdnsettings** WebSocket API interface.

### p25refsettings

P25Reflector connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 41000,
  "callsign": "",
  "tg_id": 123,
  "reroute_enabled": 1,
  "reroute_tg_id": 9,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "",
  "port": 41000,
  "callsign": "",
  "tg_id": 123,
  "reroute_enabled": 1,
  "reroute_tg_id": 9,
  "keepalive_interval_sec": 1,
  "rx_timeout_sec": 10,
  "conn_retry_interval_sec": 1
}
```

### dapnetsettings

DAPNET connector settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "modem_mode": 0,
  "server_host": "",
  "port": 43434,
  "callsign": "",
  "auth_key": "",
  "rx_timeout_sec": 65,
  "conn_retry_interval_sec": 65,
  "allow_in_bg": 0,
  "tx_time": 0,
  "decode_rot1": 1,
  "only_allowed_rics": 0,
  "allowed_ric1": 0,
  "allowed_ric2": 0,
  "allowed_ric3": 0,
  "allowed_ric4": 0,
  "allowed_ric5": 0,
  "allowed_ric6": 0,
  "allowed_ric7": 0,
  "allowed_ric8": 0,
  "send_callsign": "",
  "send_auth_key": ""
}
```
Response:
```json
{
  "modem_mode": 0,
  "server_host": "",
  "port": 43434,
  "callsign": "",
  "auth_key": "",
  "rx_timeout_sec": 65,
  "conn_retry_interval_sec": 65,
  "allow_in_bg": 0,
  "tx_time": 0,
  "decode_rot1": 1,
  "only_allowed_rics": 0,
  "allowed_ric1": 0,
  "allowed_ric2": 0,
  "allowed_ric3": 0,
  "allowed_ric4": 0,
  "allowed_ric5": 0,
  "allowed_ric6": 0,
  "allowed_ric7": 0,
  "allowed_ric8": 0,
  "send_callsign": "",
  "send_auth_key": ""
}
```

**send_callsign** and **send_auth_key** are only used by the browser to send
messages using the DAPNET API.

### srfipconnclientsettings

SharkRF IP Connector Client settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "192.168.3.120",
  "port": 65100,
  "id": 2161005,
  "password": "abcdefgh",
  "callsign": "",
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "server_host": "192.168.3.120",
  "port": 65100,
  "id": 2161005,
  "password": "abcdefgh",
  "callsign": "",
  "keepalive_interval_sec": 5,
  "rx_timeout_sec": 30,
  "conn_retry_interval_sec": 1
}
```

### srfipconnserversettings

SharkRF IP Connector Server settings query (get)/change (post).

See the **modemmode** interface for available *modem_mode* values.

Request (optional):
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "port": 65100,
  "password": "abcdefgh",
  "rx_timeout_sec": 30
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "tx_freq": 434000000,
  "modem_mode": 0,
  "port": 65100,
  "password": "abcdefgh",
  "rx_timeout_sec": 30
}
```

### autocal

AutoCal settings query (get)/change (post).
See the **modemmode** WebSocket API interface for available *modem_mode*
values and the **modemfreq** WebSocket API interface for information on
RX offsets.

Query (optional):
```json
{
  "rx_freq": 434000000,
  "dmr_rx_offset": 0,
  "c4fm_half_rx_offset": 0,
  "nxdn_rx_offset": 0,
  "p25_rx_offset": 0,
  "tx_freq": 434000000,
  "modem_mode": 1
}
```
Response:
```json
{
  "rx_freq": 434000000,
  "dmr_rx_offset": 0,
  "c4fm_half_rx_offset": 0,
  "nxdn_rx_offset": 0,
  "p25_rx_offset": 0,
  "tx_freq": 434000000,
  "modem_mode": 1
}
```

### info

Allows you to query (get) general information about the device.

*hwver* is the device hardware version.
*frwver* is the device firmware version.
*csdver* is the callsign database version. It's an empty string if the database
is not available.
*subver* is the device subversion.
*blver* is the bootloader version.
*blupgradeneeded* is 1 if the bootloader can be upgraded.
*blupgradecanstart* is 1 if the bootloader upgrade process can be started.
*uid* is the device unique ID in hexadecimal.

*dstdate1* and *dstdate2* points to the next 2 days from the next
365 days when the timezone offset will change because of the daylight
saving time. *dstoffset1* and *dstoffset2* contains the offsets to
change at the given timestamps.

Query (optional):
```json
{
  "dstdate1": 1509238800,
  "dstdate2": 1521939600,
  "dstoffset1": -3600,
  "dstoffset1": -7200
}
```
Response:
```json
{
  "hwver": "1.0",
  "frwver": "0001",
  "csdver": "201804051038",
  "subver": "433",
  "blver": "0001",
  "uid": "abcdef"
}
```

### cpsettings

Config profile settings query (get)/change (post).

*reboot* is 1 if the active config profile has been changed and openSPOT3 is
rebooting.
*init_ran* is set to 1 if the initial setup has been completed at least once in
the past.
Define *cp_name1*, *cp_name2* and so on to rename the config profile.
Set *cp_copyfrom* and *cp_copyto* if you want to copy config profiles.
*active_cp_initialized* is 1 if the currently active config profile is
initialized.
The array *cp_names* contain each config profile's name.
*timeoutchange_cp* is the profile number which should be activated
after *rxtimeout_sec* seconds.
*rxtimeout_mode* can be 0 (modem) or 1 (modem or network).

Query (optional):
```json
{
  "init_ran": 1,
  "active_cp": 0,
  "timeoutchange_cp": 0,
  "rxtimeout_sec": 0,
  "rxtimeout_mode": 0,
  "cp_name1": "slot1",
  "cp_name2": "slot2",
  "cp_name3": "slot3",
  "cp_name4": "slot4",
  "cp_name5": "slot5",
  "cp_copyfrom": 0,
  "cp_copyto": 0
}
```
Response:
```json
{
  "reboot": 0,
  "init_ran": 1,
  "active_cp": 0,
  "timeoutchange_cp": 0,
  "rxtimeout_sec": 0,
  "rxtimeout_mode": 0,
  "active_cp_hostname": "openspot3",
  "active_cp_initialized": 1,
  "cp_names": ["slot1", "slot2", "slot3", "slot4", "slot5"]
}
```

### config-export

If you want to export config profile settings, you can send a post request to
this interface. Set *profile_nr* to the profile slot number you want to export.
You can request settings in chunks. The number of available chunks is in
*chunk_count* (this can also be requested with a get request). Each chunk is
represented in string of hexadecimal character pairs. *config_size* is the
count of valid config bytes in the file. The whole config's CRC is returned in
*config_crc*. Note that *chunk* data in this example is truncated.

Set *passinc* to 1 if you want to include all passwords in the exported
configuration profile data.

Request (optional):
```json
{
  "profile_nr": 0,
  "chunk_nr": 0,
  "passinc": 0
}
```
Response:
```json
{
  "config_size": 1458,
  "config_crc": "a9fd8832",
  "chunk_count": 3,
  "chunk_nr": 0,
  "chunk": "982443abcdef"
}
```

### config-import

If you want to import config profile settings, you can send a request to this
interface.

*profile_nr* is the profile number where the imported config will be stored.
*config_size* is the count of valid config bytes in the file.
*chunk_size* is the length which this interface waits *chunk* data in string
of hexadecimal character pairs.
*chunk_count* is the total number of *chunks* needed for a successful import.
*active_cp_hostname* is the hostname of openSPOT3. This is always returned
because it may be changed after a successful import - in this case the caller
knows on what address openSPOT3 will start listening on after the device
reboots. Note that *chunk* data in this example is truncated.
*reboot* is 1 if the device is rebooting (which happens when the currently
active configuration profile slot is the destination slot).

*status* can be the following:

- 0: CONFIGAREA_IMPORT_STATUS_INITIALIZED
- 1: CONFIGAREA_IMPORT_STATUS_NEED_MORE_CHUNKS
- 2: CONFIGAREA_IMPORT_STATUS_SUCCESS
- 3: CONFIGAREA_IMPORT_STATUS_STRUCT_VERSION_MISMATCH
- 4: CONFIGAREA_IMPORT_STATUS_CRC_ERR
- 5: CONFIGAREA_IMPORT_STATUS_INVALID_FILE
- 6: CONFIGAREA_IMPORT_STATUS_FAIL

*status* will always be 0 when calling this interface without query parameters.

Query (optional):
```json
{
  "profile_nr": 0,
  "config_size": 1458,
  "config_crc": "a9fd8832",
  "chunk_nr": 0,
  "chunk": "982443abcdef"
}
```
Response:
```json
{
  "chunk_size": 1024,
  "chunk_count": 3,
  "active_cp_hostname": "openspot3",
  "status": 0,
  "reboot": 0
}
```

### netstate

Allows you to query (get) the actual network parameters.

*ssid* is the SSID of the active Wi-Fi connection.
*bssid* is the BSSID of the active Wi-Fi connection.
*chnr* is the channel number of the active Wi-Fi connection.
*ip* is the current IP address of the openSPOT3.
*mask* is the currently used network mask.
*gw* is the IP address of the currently used network gateway.
*dns1* and *dns2* are the IP addresses of the currently used DNS servers.
*regdom* is the wireless regulatory domain. Possible values are:

- 0: FCC
- 1: ETSI
- 2: TELEC

Response:
```json
{
  "ssid": "Home",
  "bssid": "",
  "chnr": 11,
  "ip": "192.168.3.100",
  "mask": "255.255.255.0",
  "gw": "192.168.3.1",
  "dns1": "192.168.3.1",
  "dns2": "0.0.0.0",
  "regdom": 1
}
```

### net

Network settings query (get)/change (post).

*reload* is 1 if the web interface should reload because of hostname or
password change.

*chkintervalsec* is the network connection check interval in seconds.

Defining the *pass* field will change the HTTP server's password.

Query (optional):
```json
{
  "hostname": "openspot3",
  "dejitter_queue_msec": 130,
  "pass": "",
  "country": "hu",
  "chkintervalsec": 60,
  "syslog_host": "log.server.com",
  "syslog_enabled": 0,
  "syslog_port": 514
}
```
Response:
```json
{
  "hostname": "openspot3",
  "dejitter_queue_msec": 130,
  "country": "hu",
  "chkintervalsec": 30,
  "syslog_host": "log.server.com",
  "syslog_enabled": 0,
  "syslog_port": 514,
  "reload": 0
}
```

### netwlscan

Wireless scan get request. Returns found SSIDs in an array of objects, or an
empty array if scan is currently ongoing. In this case, retry the request.

The value of *sec* can be "none", "wep", "wpa", "wpa2", "wpa-enterprise",
"wpa2-enterprise". *ch* is the wireless channel used by the station. *RSSI* is
in dBm.

*act_bssid* is the currently associated AP's BSSID (empty if no AP is associated).

Response:
```json
{
  "act_bssid": "",
  "results": [
    { "ssid": "Home", "bssid": "11:22:33:44:55:66", "ch": 6, "rssi": -67, "sec": "wpa2" },
    { "ssid": "Test", "bssid": "66:55:44:33:22:11", "ch": 1, "rssi": -72, "sec": "none" }
  ]
}
```

### netwl

Network wireless settings query (get)/change (post).

If *commonsettings* is 1, then the wireless configuration will be saved to all
configuration profiles.
*reload* in the response will be set to 1 when a web interface reload is neded.

Request (optional):
```json
{
  "commonsettings": 1,
  "ssid1": "Home",
  "key1": "secretpassword1",
  "bssid1": "",
  "ssid2": "Work",
  "key2": "secretpassword2",
  "bssid2": "11:22:33:44:55:66",
  "ssid3": "",
  "key3": "secretpassword3",
  "bssid3": "66:55:44:33:22:11",
  "ssid4": "",
  "key4": "",
  "bssid4": "",
  "ssid5": "",
  "key5": "",
  "bssid5": "",
  "apssid": "openSPOT3 AP",
  "apkey": "",
  "apchnr": 6,
  "start_in_ap_mode": 0,
  "ap_mode": 0
}
```
Response:
```json
{
  "commonsettings": 1,
  "ssid1": "Home",
  "key1": "secretpassword1",
  "bssid1": "",
  "ssid2": "Work",
  "key2": "secretpassword2",
  "bssid2": "11:22:33:44:55:66",
  "ssid3": "",
  "key3": "secretpassword3",
  "bssid3": "66:55:44:33:22:11",
  "ssid4": "",
  "key4": "",
  "bssid4": "",
  "ssid5": "",
  "key5": "",
  "bssid5": "",
  "apssid": "openSPOT3 AP",
  "apkey": "",
  "apchnr": 6,
  "start_in_ap_mode": 0,
  "ap_mode": 0,
  "reload": 1
}
```

### netip

Network IP settings query (get)/change (post).

*ip_config_mode* can be:

- 0: DHCP
- 1: Static IP

Request (optional):
```json
{
  "ip_config_mode": 0,
  "static_ip": "192.168.1.99",
  "static_mask": "255.255.255.0",
  "static_gw": "192.168.1.1",
  "static_dns1": "8.8.8.8",
  "static_dns2": "8.8.4.4",
  "usestaticdns": 1
}
```
Response:
```json
{
  "ip_config_mode": 0,
  "static_ip": "192.168.1.99",
  "static_mask": "255.255.255.0",
  "static_gw": "192.168.1.1",
  "static_dns1": "8.8.8.8",
  "static_dns2": "8.8.4.4",
  "usestaticdns": 1
}
```

### netmac

Network MAC settings query (get)/change (post).

Request (optional):
```json
{
  "mac": "11:22:33:44:55:66"
}
```
Response:
```json
{
  "mac": "11:22:33:44:55:66",
  "client": "66:55:44:33:22:11",
  "orig": "11:11:11:22:22:22",
  "ap": "11:11:11:33:33:33"
}
```

### nettraffic

Network traffic counters reset (post). If *reset* is set to 1, then the
traffic counters will be set to 0.

Request (optional):
```json
{
  "reset": 1
}
```

### netntp

NTP settings query (get)/change (post).

If *syncneeded* is set to 1 then an NTP sync will be initiated even if no other
fields are modified.

Request (optional):
```json
{
  "host": "pool.ntp.org",
  "syncneeded": 1,
  "use_dhcp_server": 1
}
```
Response:
```json
{
  "host": "pool.ntp.org",
  "synced_host": "pool.ntp.org",
  "use_dhcp_server": 1
}
```

### fwu

Firmware upgrade settings query (get)/change (post).

*beta* is 1 if the device should check/download beta firmwares, not just
stable ones.
*auto* is 1 if auto firmware upgrade is turned on.
*autodata* is 1 if auto data upgrade is turned on.
*daysec* is the seconds from midnight when the firmware upgrade reboot
to bootloader should be started.
*delaysec* is the minimum delay in seconds after the last modem/network
RX or TX before the device reboots.
*chkintervalsec* is the new firmware version check interval in seconds.

If *chkneeded* is 1, the device will start checking for newer firmware
versions.
If *startfwdl* is 1, the device will start downloading the firmware upgrade.
If *startdatadl* is 1, the device will start downloading the data upgrade.
If *cancel* is 1, the ongoing firmware upgrade process is cancelled.

Request (optional):
```json
{
  "beta": 0,
  "auto": 1,
  "autodata": 1,
  "daysec": 14400,
  "delaysec": 180,
  "chkintervalsec": 3600,
  "chkneeded": 0,
  "startfwdl": 0,
  "startdatadl": 0,
  "cancel": 0
}
```
Response:
```json
{
  "beta": 0,
  "auto": 1,
  "autodata": 1,
  "daysec": 14400,
  "delaysec": 180
}

### spksettings

Voice announcement settings query (get)/change (post).

*shortbm* is 1 if shortened BrandMeister announcements are needed.
*shortprofile* is 1 if shortened profile announcements are needed.
*net_state_int_sec* is the net state announcement interval in seconds.

Request (optional):
```json
{
  "enabled": 1,
  "shortbm": 0,
  "shortprofile": 0,
  "remote_only": 0,
  "host": "spk.sharkrf.com",
  "port": 65200,
  "net_state_int_sec": 30,
  "src_id": 9998,
  "prof_id": 9000,
  "con_id": 9998,
  "ip_id": 9001
}
```
Response:
```json
{
  "enabled": 1,
  "shortbm": 0,
  "shortprofile": 0,
  "remote_only": 0,
  "host": "spk.sharkrf.com",
  "port": 65200,
  "net_state_int_sec": 30,
  "src_id": 9998,
  "prof_id": 9000,
  "con_id": 9998,
  "ip_id": 9001
}
```

### locationsettings

Location settings query (get)/change (post).

Request (optional):
```json
{
  "latitude": "0.0",
  "longitude": "0.0",
  "height_agl": 0,
  "height_asl": 0,
  "name": ""
}
```
Response:
```json
{
  "latitude": "0.0",
  "longitude": "0.0",
  "height_agl": 0,
  "height_asl": 0,
  "name": ""
}
```

### dmrsettings

General DMR settings query (get)/change (post).

If *no_srvinband* is set to 1, in-band DMR data like GPS position or talker
alias will not be sent to the network.
If *no_inband* is set to 1, in-band DMR data like GPS position or talker
alias will not be sent to the modem.
If *gen_talker_alias_to_modem* is set to 1, a talker alias will be generated to
the stream sent to the modem based on the information available in the DMR ID
database.
If *force_src_id* is set to other than 0, then the source ID of all DMR calls
received by the modem will be replaced with this ID.
*tglist* is the preferred talkgroup list. Valid values are: default and tgif.
The *default_cross_mode_src_id* will be used as the source ID for DMR calls in
cross modem modes if the original source callsign can't be parsed to a valid
DMR ID.
If *allow_only_ids_as_callsigns* is set to 1, then only parseable DMR IDs are
allowed as source callsigns in cross modem modes.

Request (optional):
```json
{
  "no_srvinband": 0,
  "force_talker_alias": "abcd",
  "no_inband": 0,
  "gen_talker_alias_to_modem": 0,
  "force_src_id": 0,
  "cc": 1,
  "echo_id": 9999,
  "tglist": "default",
  "default_cross_mode_src_id": 0,
  "allow_only_ids_as_callsigns": 1
}
```
Response:
```json
{
  "no_srvinband": 0,
  "force_talker_alias": "abcd",
  "no_inband": 0,
  "gen_talker_alias_to_modem": 0,
  "force_src_id": 0,
  "cc": 1,
  "echo_id": 9999,
  "tglist": "default",
  "default_cross_mode_src_id": 0,
  "allow_only_ids_as_callsigns": 1
}
```

### dstarsettings

General D-STAR settings query (get)/change (post).

If *cross_mode_def_cs* is set then it will be used as the source callsign for
cross modem mode calls.

Request (optional):
```json
{
  "echo_callsign": "       E",
  "cross_mode_def_cs": "",
  "transmit_rx_confirmation": 1,
  "dtmf_automute_cmds": 1
}
```
Response:
```json
{
  "echo_callsign": "       E",
  "cross_mode_def_cs": "",
  "transmit_rx_confirmation": 1,
  "dtmf_automute_cmds": 1
}
```

### c4fmsettings

General C4FM settings query (get)/change (post).

If *cross_mode_def_cs* is set then it will be used as the source callsign for
cross modem mode calls.

If *only_rx_with_dgid_en* is 1, then the modem only processes C4FM calls with
DGID *only_rx_with_dgid*.

If *force_dgid_to_modem_en* is 1, all C4FM frames sent to the modem will have
DGID *force_dgid_to_modem* set.

If *force_dgid_to_net_en* is 1, all C4FM frames sent to the network will have
DGID *force_dgid_to_net* set.

Request (optional):
```json
{
  "allow_data_calls_to_net": 0,
  "ignore_wiresx_btn_cmds": 0,
  "disable_name_upcase": 0,
  "no_wiresx_conn_msgs": 0,
  "hide_profiles_in_wiresx_all_reply": 0,
  "dtmf_automute_cmds": 1,
  "dtmf_pcode": "*",
  "dtmf_gcode": "#",
  "transmit_rx_confirmation": 1,
  "cross_mode_def_cs": "",
  "echo_dgid": 99,
  "only_rx_with_dgid_en": 0,
  "only_rx_with_dgid": 12,
  "force_dgid_to_modem_en": 0,
  "force_dgid_to_modem": 12,
  "force_dgid_to_net_en": 0,
  "force_dgid_to_net": 12
}
```
Response:
```json
{
  "allow_data_calls_to_net": 0,
  "ignore_wiresx_btn_cmds": 0,
  "disable_name_upcase": 0,
  "no_wiresx_conn_msgs": 0,
  "hide_profiles_in_wiresx_all_reply": 0,
  "dtmf_automute_cmds": 1,
  "dtmf_pcode": "*",
  "dtmf_gcode": "#",
  "transmit_rx_confirmation": 1,
  "cross_mode_def_cs": "",
  "echo_dgid": 99,
  "only_rx_with_dgid_en": 0,
  "only_rx_with_dgid": 12,
  "force_dgid_to_modem_en": 0,
  "force_dgid_to_modem": 12,
  "force_dgid_to_net_en": 0,
  "force_dgid_to_net": 12
}
```

### nxdnsettings

General NXDN settings query (get)/change (post).

The *default_cross_mode_dstid* will be used as the destination ID in cross modem
modes if the original destination ID is not valid as an NXDN ID. You can force
this ID's usage by setting *always_use_cross_mode_dstid* to 1.
The *default_cross_mode_srcid* will be used as the source ID in cross modem
modes if the original source ID is not valid as an NXDN ID. You can force
this ID's usage by setting *always_use_cross_mode_srcid* to 1.
If *only_valid_callsign_ids* is set to 1, then only parseable NXDN IDs are allowed
as source callsigns in cross modem modes.

Request (optional):
```json
{
  "ran": 0,
  "echo_id": 9999,
  "default_cross_mode_dstid": 9,
  "always_use_cross_mode_dstid": 1,
  "default_cross_mode_srcid": 9998,
  "always_use_cross_mode_srcid": 0,
  "only_valid_callsign_ids": 0
}
```
Response:
```json
{
  "ran": 0,
  "echo_id": 9999,
  "default_cross_mode_dstid": 9,
  "always_use_cross_mode_dstid": 1,
  "default_cross_mode_srcid": 9998,
  "always_use_cross_mode_srcid": 0,
  "only_valid_callsign_ids": 0
}
```

### p25settings

General P25 settings query (get)/change (post).
The *default_cross_mode_src_id* will be used as the source ID for P25 calls in
cross modem modes if the original source callsign can't be parsed to a valid
P25 ID.
If *allow_only_ids_as_callsigns* is set to 1, then only parseable P25 IDs are
allowed as source callsigns in cross modem modes.

Request (optional):
```json
{
  "nac": 0,
  "echo_id": 9999,
  "default_cross_mode_src_id": 9,
  "allow_only_ids_as_callsigns": 0
}
```
Response:
```json
{
  "nac": 0,
  "echo_id": 9999,
  "default_cross_mode_src_id": 9,
  "allow_only_ids_as_callsigns": 0
}
```

### locksettings

Callsign/CCS7 ID lock settings query (get)/change (post).

Request (optional):
```json
{
  "id1": 2161005,
  "callsign1": "HA2NON",
  "id2": 2161006,
  "callsign2": "HG1MA",
  "id3": 0,
  "callsign3": ""
}
```
Response:
```json
{
  "id1": 2161005,
  "callsign1": "HA2NON",
  "id2": 2161006,
  "callsign2": "HG1MA",
  "id3": 0,
  "callsign3": ""
}
```

### modemfreq

RX, TX frequency, TX power, RX frequency offset query (get)/change (post).
RX offsets are in Hz and their range can be -500 to 500.

Request (optional):
```json
{
  "rx_frequency": 433450000,
  "dmr_rx_offset": 0,
  "dmr_cc": 1,
  "c4fm_half_rx_offset": 0,
  "nxdn_rx_offset": 0,
  "nxdn_ran": 0,
  "p25_rx_offset": 0,
  "p25_nac": 0,
  "tx_frequency": 433450000,
  "tx_power_percent": 100,
  "save_to_all_connectors": 1
}
```
Response:
```json
{
  "rx_frequency": 433450000,
  "dmr_rx_offset": 0,
  "dmr_cc": 1,
  "c4fm_half_rx_offset": 0,
  "nxdn_rx_offset": 0,
  "nxdn_ran": 0,
  "p25_rx_offset": 0,
  "p25_nac": 0,
  "tx_frequency": 433450000,
  "pocsag_freq_used": 0,
  "tx_power_percent": 100
}
```

*pocsag_freq_used* is 1 if the currently active modem mode is POCSAG.

### modemcwid

Modem CW ID settings query (get)/change (post).

Request (optional):
```json
{
  "enabled": 1,
  "modulate": 0,
  "cwid": "HA2NON",
  "wpm": 25,
  "interval_sec": 600,
  "delay_sec": 30
}
```
Response:
```json
{
  "enabled": 1,
  "modulate": 0,
  "cwid": "HA2NON",
  "wpm": 25,
  "interval_sec": 600,
  "delay_sec": 30
}
```

### modemmode

Current modem mode query (get)/change (post).

*mode* can be:

- 0: Idle
- 1: DMR
- 2: D-STAR
- 3: C4FM
- 4: C4FM half deviation mode
- 5: NXDN
- 6: P25
- 7: POCSAG

Request (optional):
```json
{
  "mode": 0
}
```
Response:
```json
{
  "mode": 0
}
```

### modemmodulation

Modem modulation mode query (get)/change (post).

*modulation_mode* can be:

- 0: 2FSK
- 1: 2FSK Raised Cosine
- 2: 4FSK
- 3: 4FSK Raised Cosine

Request (optional):
```json
{
  "modulation_mode": 0,
  "inner_deviation_hz": 648
}
```
Response:
```json
{
  "modulation_mode": 0,
  "inner_deviation_hz": 648
}
```

### modemcal

Modem calibration settings query (get)/change (post).

Request (optional):
```json
{
  "auto_calibration": 1,
  "recalibrate_temp_diff_in_celsius": 10,
  "temp_read_interval_in_sec": 10,
  "temp_read_delay_in_sec_after_sync_lost": 10,
  "quick_calibrate_delay_in_sec_after_sync_lost": 10
}
```
Response:
```json
{
  "auto_calibration": 1,
  "recalibrate_temp_diff_in_celsius": 10,
  "temp_read_interval_in_sec": 10,
  "temp_read_delay_in_sec_after_sync_lost": 10,
  "quick_calibrate_delay_in_sec_after_sync_lost": 10
}
```

### modemother

Other modem settings query (get)/change (post).

If *reset_settings* is set to 1 then the default modem configuration will be
restored.
*agc_auto* and *external_vco* fields are booleans.

Request (optional):
```json
{
  "reset_settings": 0,
  "bclo_dbm": -80,
  "ignore_rx_after_tx_ms": 0,
  "sensitivity_level": 6,
  "filter_gain": 0,
  "agc_auto": 0,
  "agc_low_threshold_dbm": -50,
  "agc_high_threshold_dbm": -80,
  "external_vco": 0,
  "call_hang_time_ms": 3000
}
```
Response:
```json
{
  "bclo_dbm": -80,
  "ignore_rx_after_tx_ms": 0,
  "sensitivity_level": 6,
  "filter_gain": 0,
  "agc_auto": 0,
  "agc_low_threshold_dbm": -50,
  "agc_high_threshold_dbm": -80,
  "external_vco": 0,
  "call_hang_time_ms": 3000
}
```

### quickcall

If you want to request a quick call to the network, you can send a request to
this interface. If the currently active connector does not support the quick
call feature, the response code will be 400 Bad Request.

If *ignore_call_from_4000* is set to 1, then the openSPOT3 ignores (mutes) the
first call coming from ID 4000.

Request (optional):
```json
{
  "dst_id": 4000,
  "private_call": 0,
  "tdma_channel": 0,
  "ignore_call_from_4000": 1
}
```

The response of the interface for every request is the following:
```json
{
  "autodisconnect": 1,
  "slots": [[216,0,0],[2161,0,0],[2162,0,0]...]
}
```

*autodisconnect* returns the previously stored *autodisconnect* setting. This
settings is only stored by the device, it won't do an auto disconnect before a
quick call even if it is set to 1.

*slots* is an array of the stored quick call shortcut slots. Each element is an
array in the format *[id, privatecall, tdmachannel]*.

If you want to change the *autodisconnect* setting, send the following request:

```json
{
  "autodisconnect": 0
}
```

If you want to change a stored shortcut slot, send the following request:

```json
{
  "slot_nr": 0,
  "save_id": 2165,
  "private_call": 0,
  "tdma_channel": 0
}
```

### initsettings

Init setup settings query (get)/change (post). Returns already stored
settings.

If *finish* is 1 in the query, then already stored init settings will
be copied to all needed places in the config. If *commonsettings* in the
*net-wl* interface is 1, then the settings will be copied to all configuration
profiles.
If openSPOT3 will reconnect to the Wi-Fi network after *finish* is set to 1 in
the query, *reconnect* is set to 1 in the response.

Request (optional):
```json
{
  "country": "us",
  "ssid": "bombegy",
  "key": "abcd1234",
  "finish": 1
}
```
Response:
```json
{
  "country": "us",
  "reconnect": 1
}
```

### bmmsettings

BrandMeister manager settings query (get)/change (post). Returns already
stored settings.

Request (optional):
```json
{
  "apikey": "abcdef"
}
```
Response:
```json
{
  "apikey": "abcdef"
}
```

### qs-cs

Quick setup callsign and/or IDs. Settings will be copied to all needed places
in the config. Returns already stored settings.

Request (optional):
```json
{
  "callsign": "HA2NON",
  "dmr_id": 2161005,
  "nxdn_id": 61005
}
```
Response:
```json
{
  "callsign": "HA2NON",
  "dmr_id": 2161005,
  "nxdn_id": 61005
}
```

### led

You can override the LED with this interface. The RGB values are in percent.
*led* can be "app" or "pwr".

Request (optional):
```json
{
  "led": "app",
  "override": 1,
  "r": 25,
  "g": 0,
  "b": 25
}
```
Response:
```json
{
  "override": 1
}
```

### csdlookup

You can query information from the openSPOT3's callsign-ID (CSD) database.

Request:
```json
{
  "id": 2161005,
  "type": 0,
  "callsign": ""
}
```
Response:
```json
{
  "result": 0,
  "dmrids": [2161005, 2161028],
  "dmrtgids": [],
  "nxdnids": [61005],
  "callsign": "HA2NON",
  "name": "Norbert",
  "city": "",
  "state": "",
  "country": "hu"
}
```

The request can only contain either the ID or the callsign. If both fields are
present then the result will be a bad request.

*result* can be the following:

-  0: Success
- -1: ID/callsign not found in the database
- -2: No CSD database is present on the device
- -3: Error during the database query

*type* is the ID type (this only matters if the request is made for an ID,
not a callsign). Valid values are:

- 0: DMR
- 1: NXDN
- 2: DMR talkgroup
- 3: DMR TGIF Network talkgroup

### pocsagsettings

POCSAG settings query (get)/change (post). Returns already stored settings.

Request:
```json
{
  "freq": 439987500,
  "bitrate_mode": 0,
  "txdelay_sec": 10,
  "ts_nooverride": 0,
  "timeslots": 65535
}
```
Response:
```json
{
  "freq": 439987500,
  "bitrate_mode": 0,
  "txdelay_sec": 10,
  "ts_nooverride": 0,
  "timeslots": 65535
}
```

*bitrate_mode* can be the following:

- 0: 1200bps
- 1: 2400bps
- 2: 512bps

*timeslots* is a 16 bit value. Each bit represents a timeslot, so for example
the value 32768 means timeslot F is allowed, and others are not.

### pocsagmsg

POCSAG message queue add/empty (post).

Request:
```json
{
  "dst_ric": 161005,
  "msg": "beer",
  "alert_type": 0,
  "empty": 0
}
```
Response:
```json
{
  "empty_result": 0,
  "add_result": 1
}
```

If only a message send is needed, field *empty* is not needed. *add_result* is
1 if add to the queue was successful.

If you want to empty the POCSAG message queue, set *empty* to 1. If emptying
the queue was successful, *empty_result* will be 1.

*alert_type* can be the following:

- 0: Not an alert message.
- 1: Alert without message. In this case the *msg* field will be ignored.
- 2: Alert with message.

### aprssettings

APRS settings query (get)/change (post). Returns already stored settings.

Request (optional):
```json
{
  "server_host": "rotate.aprs2.net",
  "port": 14580,
  "callsign": "HA2NON",
  "rx_timeout_sec": 65,
  "conn_retry_interval_sec": 5,
  "location_symbol": "\\&",
  "location_comment": "SharkRF openSPOT3",
  "location_send_enabled": 1,
  "allow_in_bg": 1,
  "send_to_pocsag_ric": 0,
  "allow_dstar_loc_fwd": 1,
  "force_dstar_loc_ssid_en": 0,
  "force_dstar_loc_symbol_en": 0,
  "force_dstar_loc_comment_en": 0,
  "allow_c4fm_loc_fwd": 1,
  "force_c4fm_loc_ssid_en": 0,
  "force_c4fm_loc_symbol_en": 0,
  "force_c4fm_loc_comment_en": 0,
  "force_dstar_loc_ssid": 7,
  "force_dstar_loc_symbol": "/[",
  "force_dstar_loc_comment": "SharkRF openSPOT3",
  "force_c4fm_loc_ssid": 7,
  "force_c4fm_loc_symbol": "/[",
  "force_c4fm_loc_comment": "SharkRF openSPOT3"
}
```
Response:
```json
{
  "server_host": "rotate.aprs2.net",
  "port": 14580,
  "callsign": "HA2NON",
  "rx_timeout_sec": 65,
  "conn_retry_interval_sec": 5,
  "location_symbol": "\\&",
  "location_comment": "SharkRF openSPOT3",
  "location_send_enabled": 1,
  "allow_in_bg": 1,
  "send_to_pocsag_ric": 0,
  "allow_dstar_loc_fwd": 1,
  "force_dstar_loc_ssid_en": 0,
  "force_dstar_loc_symbol_en": 0,
  "force_dstar_loc_comment_en": 0,
  "allow_c4fm_loc_fwd": 1,
  "force_c4fm_loc_ssid_en": 0,
  "force_c4fm_loc_symbol_en": 0,
  "force_c4fm_loc_comment_en": 0,
  "force_dstar_loc_ssid": 7,
  "force_dstar_loc_symbol": "/[",
  "force_dstar_loc_comment": "SharkRF openSPOT3",
  "force_c4fm_loc_ssid": 7,
  "force_c4fm_loc_symbol": "/[",
  "force_c4fm_loc_comment": "SharkRF openSPOT3"
}
```

### aprsmsgsend

APRS message queue add (post). If you set *send_cancel* to 1 then the currently
ongoing message send will be cancelled.

Request:
```json
{
  "addressee": "HG1MA",
  "msg": "beer",
  "send_unconfirmed": 0,
  "send_cancel": 0
}
```
Response:
```json
{
  "id": 123
}
```

*id* is the queued (or cancelled) message's ID, or -1 if message adding to the
queue (or cancel) was unsuccessful.

### gainsettings

Transcode gain settings query (get)/change (post). Returns already stored
settings.

Request:
```json
{
  "dmr2dstar": 0,
  "dmr2c4fm": 0,
  "dmr2nxdn": 0,
  "dstar2dmr": 0,
  "dstar2c4fm": 0,
  "dstar2nxdn": 0,
  "c4fm2dmr": 0,
  "c4fm2dstar": 0,
  "c4fm2nxdn": 0,
  "nxdn2dmr": 0,
  "nxdn2dstar": 0,
  "nxdn2c4fm": 0,
}
```
Response:
```json
{
  "dmr2dstar": 0,
  "dmr2c4fm": 0,
  "dmr2nxdn": 0,
  "dstar2dmr": 0,
  "dstar2c4fm": 0,
  "dstar2nxdn": 0,
  "c4fm2dmr": 0,
  "c4fm2dstar": 0,
  "c4fm2nxdn": 0,
  "nxdn2dmr": 0,
  "nxdn2dstar": 0,
  "nxdn2c4fm": 0,
}
```

### miscsettings

Miscellaneous settings query (get)/change (post). Returns already stored
settings.

Request:
```json
{
  "dark_mode": 0,
  "pwrsaving": 0,
  "turn_off_if_no_charge": 0,
  "no_fast_charge": 0,
  "led_app_pwr_percent": 50,
  "led_power_pwr_percent": 50
}
```
Response:
```json
{
  "dark_mode": 0,
  "pwrsaving": 0,
  "turn_off_if_no_charge": 0,
  "no_fast_charge": 0,
  "led_app_pwr_percent": 50,
  "led_power_pwr_percent": 50
}
```

### beepersettings

Beeper settings query (get)/change (post). Returns already stored settings.
The openSPOT3 emits a find device signal and blinks LEDs when *find_device* is
set to 1.

Request:
```json
{
  "enabled": 1,
  "vol_percent": 100,
  "morse_wpm": 20,
  "batlow_enabled": 1,
  "connect_enabled": 1,
  "profile_on_startup_enabled": 0,
  "profile_name_on_startup_enabled": 0,
  "modem_mode_change_enabled": 0,
  "find_device": 0
}
```
Response:
```json
{
  "enabled": 1,
  "vol_percent": 100,
  "morse_wpm": 20,
  "batlow_enabled": 1,
  "connect_enabled": 1,
  "profile_on_startup_enabled": 0,
  "profile_name_on_startup_enabled": 0,
  "modem_mode_change_enabled": 0
}
```

### beep

A custom beep can be emitted using the openSPOT3's beeper. If *morse* is set,
then the openSPOT3 will play the given string in morse code. *hz* and
*length_ms* are ignored in this case.

Request:
```json
{
  "hz": 1000,
  "length_ms": 500,
  "morse": ""
}
```
Response:
```json
{
  "hz": 1000,
  "length_ms": 500
}
```

### rtcsettings

RTC settings query (get)/change (post). Returns already stored settings.
*wakeup_at* and *pdown_at* fields represent seconds after midnight (local
time).

Request:
```json
{
  "wakeup_enabled": 0,
  "wakeup_at": 100,
  "wakeup_cp_enabled": 0,
  "wakeup_cp": 0,
  "pdown_enabled": 0,
  "pdown_at": 200
}
```
Response:
```json
{
  "wakeup_enabled": 0,
  "wakeup_at": 100,
  "wakeup_cp_enabled": 0,
  "wakeup_cp": 0,
  "pdown_enabled": 0,
  "pdown_at": 200
}
```
