# SSDP

SSDP (Simple Service Discovery Protocol) is the first layer in the
[UPnP (Universal Plug and Play) Device Architecture](http://www.upnp.org/specs/arch/UPnP-arch-DeviceArchitecture-v1.1.pdf).
It allows user devices like smartphones or tablets (called _control points_) to
search for other devices or services of interest on the network.

SSDP is also used as part of other protocols separately from the whole UPnP
stack. One such protocol
is [DIAL (DIscovery And Launch)](http://www.dial-multiscreen.org/), which allows
second-screen devices like smartphones or tablets to discover and launch
applications on first-screen devices like smart TVs, set-top boxes and game
consoles.

Another example is
[HbbTV 2.0 (Hybrid broadcast broadband TV)](https://www.hbbtv.org/resource-library/),
which extends DIAL as to discover HbbTV devices and launch HbbTV applications.

[Fraunhofer FOKUS](https://www.fokus.fraunhofer.de/fame) has
[proposed](https://github.com/google/physical-web/blob/master/documentation/ssdp_support.md)
the use of SSDP to advertise and find URLs in a local area network, as part of
the [Physical Web Project](https://github.com/google/physical-web).


## Design and Specification

SSDP allows uPnP _root devices_ (that offer services like TVs, printers, etc.)
to advertise services to control points on the network. It also allows control
points to search for devices or services of interest at any time. SSDP specifies
the messages exchanged between control points and root devices.

SSDP advertisements contain specific information about the service or device,
including its type and a unique identifier.  SSDP messages adopt the header field
format of HTTP 1.1.  However, the rest of the protocol is not based on HTTP
1.1, as it uses UDP instead of TCP and it has its own processing rules.

The following sequence diagram shows the SSDP message exchange between a control
point and root device.

![](images/ssdp.png)


### Message Flow

1. The root device advertises itself on the network by sending a `NOTIFY`
   message of type `ssdp:alive` for each service it offers to the multicast
   address `239.255.255.250:1900` with all the information needed to access that
   service.  Control points listening on the multicast address receive the
   message and check the service type (`NT`) header to determine if it is
   relevant or not.

   To obtain more information, the control point makes an HTTP request to the
   device description URL provided in the `LOCATION` header of the `NOTIFY`
   message. The device description is a XML document that contains information
   about the device like its friendly name and capabilities, as well as
   information about each of the services it offers.

1. A control point can search for root devices at any time by sending a
   `M-SEARCH` query to the same multicast address. The query contains the search
   target (`ST`) header specifying the service type the control point wants.
   All devices listing to the multicast address will receive the query.

1. When a root device receives a query, it checks the `ST` header against the
   list of services offered.  If there is a match it replies with a unicast
   `M-SEARCH` response to the control point that sent the query.

1. When a service is no longer available, the root device multicasts a `NOTIFY`
   message of type `ssdp:byebye` with `ST` set to the service type.  Control
   points can remove the service from any caches.

## Evaluation

This section evaluates SSDP as a discovery protocol for the Open Screen Protocol
according to several functional and non-functional requirements.

### Functional Requirements: Presentation API

For the Presentation API, the requirement is the ability to "Monitor Display
Availability" by a controlling user agent (_controller_) as described
in
[6.4 Interface PresentationAvailability](https://w3c.github.io/presentation-api/#interface-presentationavailability).

The entry point in the Presentation API to monitor display availability is the
[PresentationRequest](https://w3c.github.io/presentation-api/#interface-presentationrequest)
interface. The algorithm
[_monitoring the list of available presentation displays_](https://w3c.github.io/presentation-api/#dfn-monitor-the-list-of-available-presentation-displays)
is used in
[PresentationRequest.start()](https://w3c.github.io/presentation-api/#dom-presentationrequest-start)
and in
[PresentationRequest.getAvailability()](https://w3c.github.io/presentation-api/#dom-presentationrequest-getavailability).
Only presentation displays that can open at least one of the URLs passed as
input in the PresentationRequest constructor are considered available for that
request. There are at least three ways SSDP can be used to monitor display
availability for the Presentation API:

#### Method 1

Similar to of SSDP discovery in DIAL. The main steps are listed below:

1. The presentation display device advertises using SSDP the presentation
   receiver service when it is connected to the network with the service type
   `urn:openscreen-org:service:openscreenreceiver:1`.  The `ssdp:alive` message
   contains a `LOCATION` header which points to the XML device description,
   which includes the friendly name, device capabilities, and other device data.

    ```
    NOTIFY * HTTP/1.1
    HOST: 239.255.255.250:1900
    CACHE-CONTROL: max-age = 1800 [response lifetime]
    LOCATION: http://192.168.0.123:8080/desc.xml [device description URL]
    NTS: ssdp:alive
    SERVER: OS/version UPnP/1.0 product/version
    USN: XXX-XXX-XXX-XXX [UUID for device]
    NT: urn:openscreen-org:service:openscreenreceiver:1
    ```

1. The controller starts to monitor display availability by sending an SSDP
   `M-SEARCH` query with the service type
   `urn:openscreen-org:service:openscreenreceiver:1` and waits for responses from
   presentation displays.  The controller should wait for `ssdp:alive` and
   `ssdp:byebye` messages on the multicast address to keep its list of available
   displays up-to-date when a new display is connected or an existing one
   is disconnected.

    ```
    M-SEARCH * HTTP/1.1
    HOST: 239.255.255.250:1900
    MAN: ssdp:discover
    MX: 2 [seconds to delay response]
    ST: urn:openscreen-org:service:openscreenreceiver:1
    ```

1. Each presentation display connected to the network running a presentation
   receiver service replies to the search request with a SSDP message similar to
   the `ssdp:alive` message.

    ```
    HTTP/1.1 200 OK
    CACHE-CONTROL: max-age = seconds until advertisement expires e.g. 2
    DATE: when response was generated
    LOCATION: http://192.168.0.123:8080/desc.xml [device description URL]
    SERVER: OS/version UPnP/1.0 product/version
    USN: XXX-XXX-XXX-XXX [UUID for device]
    ST: urn:openscreen-org:service:openscreenreceiver:1
    ```

1. When the controller receives a response from a newly connected display, it
   issues an HTTP GET request to the URL in the `LOCATION` header to get the
   device description XML.

1. The controller parses the device description XML, extracts the friendly name
   of the display and checks if the display can open one of the URLs associated
   with an existing call to `PresentationRequest.start()` or
   `PresentationRequest.getAvailability()`.  If yes, the presentation display
   will be added to the list of available displays, and the result sent back to
   pages through the Presentation API.

1. When a presentation display is disconnected it should advertise a
   `ssdp:byebye` message.

    ```
    NOTIFY * HTTP/1.1
    HOST: 239.255.255.250:1900
    NT: urn:openscreen-org:service:openscreenreceiver:1
    NTS: ssdp:byebye
    USN: XXX-XXX-XXX-XXX [UUID for device]
    ```

*Open questions:*

1. How to advertise the endpoint of the Open Screen receiver service? One
   solution is to use a HTTP header parameter in the response of the HTTP GET
   request for device description.  DIAL uses this solution to send the endpoint
   of the DIAL server in the `Application-URL` HTTP response header. Another
   solution is to extend the XML device description with a new element to define
   the endpoint.

1. How to check if the display can present a certain URL or not?  One solution
   is to extend the XML device description with new elements to allow a display
   to express its capabilities and the controller can do the check. Another
   possible solution is to ask the receiver service using the provided endpoint
   by sending the presentation URLs.

#### Method 2

This method uses only the SSDP messages without requiring the device description
XML. The presentation request URLs are sent by the controller in a new header of
the SSDP `M-SEARCH` message.  If a presentation receiver can open one of the URLs,
it responds with that URL in the search response.

The search response also adds new headers for the device friendly name and the
service endpoint.  (Note that Section 1.1.3 of the UPnP device architecture
document allows to use vendor specific headers).  The search response can still
send the device description URL in the `LOCATION` header to stay compatible with
UPnP, but controllers will ignore it.  This method is more efficient and secure
since no additional HTTP calls and XML parsing are required.

Below are the steps that illustrate this method:

1. A new presentation display that connects to the network advertises
   the following `ssdp:alive` message. The new header
   `FRIENDLY-NAME.openscreen.org` is used for the friendly name and
   `PRESENTATION-ENDPOINT.openscreen.org` for the endpoint of the receiver
   service.

    ```
    NOTIFY * HTTP/1.1
    HOST: 239.255.255.250:1900
    CACHE-CONTROL: max-age = 1800 [response lifetime]
    LOCATION: [ignored]
    NTS: ssdp:alive
    SERVER: OS/version UPnP/1.0 product/version
    USN: XXX-XXX-XXX-XXX [UUID for device]
    NT: urn:openscreen-org:service:openscreenreceiver:1
    FRIENDLY-NAME.openscreen.org: My Presentation Display
    PRESENTATION-ENDPOINT.openscreen.org: 192.168.1.100:3000
    ```
    
    [Issue #22](https://github.com/webscreens/openscreenprotocol/issues/22):
    Ensure that advertised friendly names are i18n capable

1. A controller sends the following SSDP search message to the multicast
   address. The new header `PRESENTATION-URLS.openscreen.org` allows the controller
   to send the presentation URLs to the display.

    ```
    M-SEARCH * HTTP/1.1
    HOST: 239.255.255.250:1900
    MAN: "ssdp:discover"
    MX: seconds to delay response
    ST: urn:openscreen-org:service:openscreenreceiver:1
    PRESENTATION-URLS.openscreen.org: https://example.com/foo.html, https://example.com/bar.html
    ```

    [Issue #21](https://github.com/webscreens/openscreenprotocol/issues/21):
    Investigate mechanisms to pre-filter devices by Presentation URL

1. A display that can open one of the URLs replies (unicast) with the following
   SSDP message. The new SSDP header `SUPPORTED-URLS.openscreen.org` contains the
   URLs the receiver can open from the list of the URLs
   `PRESENTATION-URLS.openscreen.org` sent in the search SSDP message.

    ```
    HTTP/1.1 200 OK
    CACHE-CONTROL: max-age = 1800 [response lifetime]
    DATE: when response was generated
    LOCATION: [ignored]
    SERVER: OS/version UPnP/1.0 product/version
    USN: XXX-XXX-XXX-XXX [UUID for device]
    ST: urn:openscreen-org:service:openscreenreceiver:1
    FRIENDLY-NAME.openscreen.org: My Presentation Display
    PRESENTATION-ENDPOINT.openscreen.org: 192.168.1.100:3000
    SUPPORTED-URL.openscreen.org: https://example.com/foo.html
    ```

1. The display sends the following SSDP message when the receiver service is no
   longer available. There are no new headers added to the `ssdp:byebye` message.

    ```
    NOTIFY * HTTP/1.1
    HOST: 239.255.255.250:1900
    NT: urn:openscreen-org:service:openscreenreceiver:1
    NTS: ssdp:byebye
    USN: XXX-XXX-XXX-XXX [UUID for device]
    ```

*Open questions*

1. SSDP messages are limited to ~1400 bytes on a typical network. Will the
   Presentation URLs and friendly names fit into message?

2. Is there a privacy issue regarding the advertisement of presentation URLs and
   friendly names to all devices on the local area network?

#### Method 3

This approach is identical to Method 2, except that presentation URLs are not
included in SSDP messages. Only the `PRESENTATION-ENDPOINT.openscreen.org`
header is added to the search response, and additional information from the
presentation receiver service (including presentation URL compatibility) is
obtained from the application level protocol implemented on that endpoint.  The
friendly name may or may not be included in advertisements, based on a tradeoff
between usability, efficiency and privacy.

### Functional Requirements: Remote Playback API

[Issue #3](https://github.com/webscreens/openscreenprotocol/issues/3): Add requirements for Remote Playback API

### Non-functional Requirements

#### Reliability

As UDP is unreliable, UPnP recommends sending SSDP messages 2 or 3 times with a
delay of few hundred milliseconds between messages.  In addition, the
presentation display must re-broadcast advertisements periodically prior to
expiration of the duration specified in the `CACHE-CONTROL` header (whose
minimum value is 1800s).

#### Latency of device discovery / device removal

New presentation displays added or removed can be immediately detected if the
controller listens to the multicast address for `ssdp:alive` and `ssdp:byebye`
messages. For search requests, the latency depends on the `MX` SSDP header which
contains the maximum wait time in seconds (and must be between 1 and 5 seconds).
SSDP responses from presentation displays should be delayed a random duration
between 0 and `MX` to balance load for the controller when it processes
responses.

#### Ease of implementation / deployment

It is very easy to implement the SSDP protocol, as it is based soley on UDP and
the messages are easy to create and parse.  Fraunhofer FOKUS offers two open
source implementations:
* [peer-ssdp](https://github.com/fraunhoferfokus/peer-ssdp) for Node.js
* [cordova plugin](https://github.com/fraunhoferfokus/cordova-plugin-hbbtv/tree/master/src/android/ssdp)
for Android

#### Security - both of the implementation, and whether it can be leveraged to enhance security of the entire protocol

SSDP considers local network as secure environment; any device on the local area
network can discover other devices or services.  Authentication and data privacy
must be implemented on the service or application level. In our case, the output
of discovery is a list of displays and friendly names, which are available to
all devices on the local area network.  If Method 2 is adopted, presentation
URLs and endpoints of presentation receiver services are also broadcast .
Additional security mechanisms can be implemented during session establishment
and communication.

[Issue #25](https://github.com/webscreens/openscreenprotocol/issues/25): Address history of uPnP exploits.

#### Privacy: what device information is exposed

The standard UPnP device description exposes parameters about the device/service
like unique identifier, friendly name, software, manufacturer, and service
endpoints. In addition, the SSDP vendor extensions proposed in Method 2
advertise presentation URLs and friendly names to all devices on the local area
network, which may expose private information in an unintended way.

#### Network efficiency

It depends on multiple factors like the number of devices in the network using
SSDP (includes devices that support DLNA, DIAL, HbbTV 2.0, etc.), the number of
services provided by each device, the interval to re-send refreshment messages
(value of `CACHE-CONTROL` header), the number of devices/applications sending
discovery messages.

#### Power efficiency

This depends on many factors including the method chosen above; Methods 2 and 3 are
better than Method 1 regarding power efficiency.  In Method 1, the controller
needs to create and send SSDP search requests, receive and parse SSDP messages,
make HTTP requests to get device descriptions and parse device description XML
to get friendly name and check capabilities.

In Methods 2 and 3, the controller needs only to create and send search requests
and receive and parse SSDP messages.

The way how controllers search for presentation displays has an impact on power
efficiency. If a controller needs to immediately react to connection and
disconnection of presentation displays, it will need to continuously receive
data on the multicast address, including all SSDP messages sent by other
controllers.  (An exception is unicast search response messages sent to other
controllers.)

If the controller needs to get only a snapshot of available displays, then it
only needs to send a search message to the multicast address and listen for
search response messages for 2-10 seconds.

[Issue #26](https://github.com/webscreens/openscreenprotocol/issues/26): Collect data regarding network and power efficiency

#### IPv4 and IPv6 support

SSDP supports IPv4 and IPv6. "Appendix A: IP Version 6 Support" of the UPnP
Device architecture document describes all details about support for IPv6.

#### Standardization status and likelihood of successful interop

SSDP is part of the UPnP device architecture. The most recent version of the
specification is
[UPnP Device Architecture 2.0](http://upnp.org/specs/arch/UPnP-arch-DeviceArchitecture-v2.0.pdf) from
February 20, 2015. On January 1, 2016, the UPnP Forum assigned their assets to the
[Open Connectivity Foundation (OCF)](https://openconnectivity.org/resources/specifications/upnp).
UPnP/SSDP is used in many products like smart TVs, printers, gateways/routers,
NAS, and PCs. According to [DLNA](https://www.dlna.org/), there are over four
billion DLNA-certified devices available on the market. SSDP is also used in
non-DLNA certified devices that support DIAL and HbbTV 2.0, including smart TVs and
digital media receivers, as well as proprietary products like
[SONOS](http://musicpartners.sonos.com/?q=docs) and
[Philips Hue](https://www.developers.meethue.com/).

[Issue #27](https://github.com/webscreens/openscreenprotocol/issues/27): Investigate uPnP licensing requirements

### Notes

* The identifiers `urn:openscreen-org:service:openscreenreceiver:1` and
  `openscreen.org` are used for illustrative purposes and may not be the
  eventual identifiers assigned for Open Screen presentation services.