#Notes on UPnP and my design

Some of this is abstracted from v1 of the UPnP Device Architecture document which can currently be found at http://upnp.org/specs/arch/UPnP-arch-DeviceArchitecture-v1.1.pdf

UPnP revolves around the concept of Devices, Services and Control Points.  Devices provide services.  Control Points use calls to services to make devices do things.  Rupture does not support control points (it doesn't need to).

A device may be logically partitioned - the base or "root" device may - as well as providing services - contain other devices that themselves provide services.  I don't think that more than one level of nesting is allowed (ie a device can't be contained within any other device except the root device).  Any device may provide zero, one or many services.

In rupture, these things are modelled as the classes UPnPDevice, UPnPRootDevice and UPnPService.  Since a root device is a special kind of device, it's defined as a subclass of UPnPDevice.

Devices and services have two important properties: Type and Version.  The UPnP forum has created specs for standard types (e.g. MediaServer) but others can be defined.  The classes in rupture only support standard types because that's all I need.  Version numbers are straight integers (no version 1.5).  Devices that contain a service at version x are expected to support versions 1 through x, not just x itself.

There are six things in UPnP that devices and/or services get involved with:

1.  Addressing.  This is the act of establishing a valid IP address for a device when it joins a network.  Rupture assumes it's running in an environment where the OS has taken care of all of this, so does not support Addressing.

2.  Discovery.  When a UPnP device starts up, it sends a series of "advertisment" messages using a protocol called SSDP.  These are sent as multicast UDP packets.  The number and content of the messages depends on whether the root device has any embedded devices, and which services are offered by the collection of devices.  There's a lot of boilerplate in the messages, the key things sent are the type and version for each device and each service offered by each device, as well as the uuid of the device (rupture assigns a uuid when any instance of UPnPDevice is created).  A URL that's used for stage 3 (Description) is also sent.

  The advertisment messages are repeated at a configurable interval, anything from 15 minutes upwards.

  When the device shuts down (assuming it can do so gracefully) it will send another series of messages cancelling the advertisments.

  The device also needs to listen for "search requests" which are sent by UPnP control points (things that request services) when they join the network (and maybe at other times?).  These search requests may ask for all root devices, all devices (root and embedded), devices or services of a specific type and version, or a device with a particular UUID.  The device must respond with information about the service(s) and device(s) that meet the request criteria, the format is similar to (and the key information the same as) the advertisments above.

  The search requests are also sent over multicast UDP, the responses are also sent over UDP but just to the IP address and port that the request came from.

  All messages are clear text and have a fairly simple, rigid format (parsing the search request just takes a handful of regular expressions).

3.  Description.  Technically this is much simpler than discovery.  By this time a control point knows that a device of interest is out there, and what services are offered.  It also has the device's Description URL.  The control point fires a standard http request to the URL and the device responds with a summary in XML format of
    - the root device
    - services offered by the root device, including URLs to retrieve a "Service Description", for Control and Eventing (see below)
    - any embedded devices and services offered by them, in the same format as the root device and service(s)
    
  The Service Description is a summary in XML format a bit like the interface description for a class.  Services have zero or more State Variables (variables) and Actions (methods).  State Variables have a data type and may have constraints on their values.  Actions have input arguments (with defined names and types) and output arguments (ditto).  For standard services defined by the UPnP forum the service description will be identical (except for XML formatting differences) across all devices offering the same service type and version.  Again service descriptions are retrieved via a standard http request to the relevant URL.
  
4.  Control.

  Control points will issue a request over http to the Control URL for a service.  The request is a stream of XML (actually SOAP format) which contains the name of the action to be invoked and the arguments to use.  The device should invoke the action and send a http response with the output parameters in XML/SOAP.  If the invocation fails it sends an error message instead.

5.  Eventing

  Control points can keep track of the state of a device by subscribing to a service.  When this happens, the device will send the current values of all State Variables to the control point.  When any state variable value changes, for any reason, the device will send a message containing the new value.  All the messages are in XML format and sent over http.
  
  Some state variables may change so frequently that they would flood the network with messages.  These variables need to be "moderated", which means update messages are only sent periodically (a few times a second) with the latest value if it has changed since the time the last message was sent.
  
  Subscriptions can be renewed by sending a similar message to the original subscription.  If a subscription is not renewed it will eventually expire.

6.  Presentation.  This is an optional URL that a standard web browser can connect to to get information about, and potentially manipulate, the device through means other than UPnP. The specs are totally relaxed about whether and how this is used, I guess because it's not really UPnP.


In summary, rupture needs to do the following:

Send multicast UDP messages (for advertisment and cancelling advertisments)  
Act as a server for multicast UDP messages (for search requests)  
Send non-multicast UDP messages (responding to search requests)  
Act as a http server (for Discovery, Control, Presentation and handling state variable subscriptions / renewals / cancellations)  
Act as a http client (sending event messages to control points when state variables change)  