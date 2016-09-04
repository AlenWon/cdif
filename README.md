Common device interconnect framework
------------------------------------

Common device interconnect framework (CDIF) is an attempt to provide an interconnect solution for Smart Home and IoT devices.

CDIF takes the assumption that gateway devices would be the central control hub for all Smart Home and IoT devices. Compared to the solution to directly control IoT devices with mobile devices, gateways have richer network I/O and protocol support, persistent connectivity and storage, more computing resource for data processings, and many other benefits. CDIF assumes itself runs on gateway, connecting to smart home and IoT devices, whether they are based on Bluetooth, ZWave, ZigBee, or IP networking protocols. After devices are discovered and connected, CDIF provide a simple set of common device management APIs to all authenticated clients to control these devices and receive event updates from them.

To take advantage of rich set of standard-based web technology and powerful Node.js ecosystem, CDIF is written in Node.js and exports a set of clean RESTful APIs for smart home and IoT devices. CDIF design tries to implement support to popular open connectivity standards, such as Bluetooth LE, ZWave, ONVIF, UPnP  and etc. within a common device management interface and unified device model to describe every one of them.

To achieve this, CDIF design is inspired by UPnP and try to define a common device model for different kinds of smart home and IoT devices. UPnP has a well defined, hybrid device model with interchangable services describing a device's capabilities. This made it look like a good start point for a common IoT device model. The application level of device profile of other popluar open connectivity standards, such as Bluetooth LE, ZWave, ONVIF etc. also can be mapped to this service oriented architecture. An example of mapping from Bluetooth LE GATT profile to CDIF's common device model can be found at [CDIF BLE manager module](https://github.com/out4b/cdif-ble-manager).

Upon device discovery process, this JSON based device model is sent to client side through CDIF's RESTful interface, thus clients web apps would know how to send action commands, get latest device states event update. By doing this, CDIF presents client side a top level device abstraction and application level profile for all devices connected to a gateway. For more information about this JSON based device model, please refer to spec/ folder in the source repository.

At the lower level, CDIF provides a set of uniformed APIs to group different types of devices into modules. Each module can manage one or more devices in same category, such as Bluetooth LE, ZWave, UPnP and etc. Theoriotically, in this design vendor's non-standard, proprietary implementations may also be plugged-in into CDIF framework as modules, and present to client side this JSON based device model. However to ensure interoperability, and also avoid the risk of unmanaged I/O which could be exposed by arbitrary implementations, proprietary implementation may need to follow open standards as much as possible and implement their device modules as sub-modules to the basic protocol modules such as ```cdif-ble-manager```, and left all I/O being managed by it.

CDIF's common device model in summary
-------------------------------------
    {
      "configId": 1,
      "specVersion": {
        "major": 1,
        "minor": 0
      },
      "device": {
        "deviceType": "urn:cdif-net:device:<deviceType>:1",
        "friendlyName": "device name",
        "manufacturer": "manufacturer name",
        "manufacturerURL": "manufacturer URL",
        "modelDescription": "device model description",
        "modelName": "device model name",
        "modelNumber": "device model number",
        "serialNumber": "device serial number",
        "UPC": "universal product code",
        "userAuth": true | false,
        "powerIndex": power consumption index,
        "devicePresentation": true | false,
        "iconList": [
          {
            "mimetype": "image/format",
            "width": "image width",
            "height": "image height",
            "depth": "color depth",
            "url": "image URL"
          }
        ],
        "serviceList": {
          "urn:cdif-net:serviceID:<serviceID>": {
            "serviceType": "urn:cdif-net:service:<serviceType>:1",
            "actionList": {
              "<actionName>": {
                "argumentList": {
                  "<argumentName>": {
                    "direction": "in  | out",
                    "retval": false | true,
                    "relatedStateVariable": "<state variable name>"
                  }
                }
              }
            },
            "serviceStateTable": {
              "<state variable name>": {
                "sendEvents": true | false,
                "dataType": "<dataType>",
                "allowedValueRange": {
                  "minimum": "",
                  "maximum": "",
                  "step": ""
                },
                "defaultValue": ""
              }
            }
          }
        },
        "deviceList": [
          "device": {
            "<embedded device list>"
          }
        ]
      }
    }

Since this model contains an abstract action call interface with arbitrary arguments definition, it would be flexible to support any kind of device API interface. By utilizing this common device model, CDIF design hopes to provide a web based common API interface for IoT devices.

In original UPnP's definitions, once device discovery is done, the returned device model would present services as URLs, and it requires additional service discovery step to resolve the full service models. Unlike this, CDIF's common device model tries to put all service models together inside device model object to present the full capabilities of a device. And the service discovery process for each protocol, if exists, is assumed to be conducted by the underlying stack. CDIF won't expose any "service discovery" framework API interface, hoping to simplify client design, and also to be better compatible with protocols which have no service discovery concept. In addition, elements such as services, arguments, state variables in CDIF's common device model are indexed by their keys for easier addressing.

But still, due to the design of underlying network protocols such as Z-Wave, it could take hours for the device to report its full capabilities. In this case, framework would progressively update device models to reflect any new capabilities reported from the network. To uncover these new device capabilities, client may need to refresh device's model by invoking ```get-spec``` RESTful API interface at different times. please refer to [cdif-openzwave](https://github.com/out4b/cdif-openzwave) for more information on this.

Features
--------
This framework now has basic support to below connectivity standards:
* [Bluetooth Low Energy](https://github.com/out4b/cdif-ble-manager)
* [ONVIF Profile S camera](https://github.com/out4b/cdif-onvif-manager)
* [Z-Wave](https://github.com/out4b/cdif-openzwave)

How to run
----------
```sh
    cd cdif
    npm install
    npm start
```
It may require root privilege to access Bluetooth interface

Summary of framework API interface:
-----------------------------------

##### Discover all devices
Start discovery process for all modules

    POST http://gateway_host_name:3049/discover
    response: 200 OK

##### Stop all discoveries
Stop all discovery processes

    POST http://gateway_host_name:3049/stop-discover
    response: 200 OK

##### Get device list
Retrieve uuid of all discovered devices. To improve security, this API won't expose the services provided by the discovered devices. The full description would be available from get-spec interface after client successfully connect to the device, which may need to provide valid JWT token if this device requires authentication.

    GET http://gateway_host_name:3049/device-list
    request body: empty
    response:
    [device-uuid1: {...}, device-uuid2: {...}]
    response: 200 OK

##### Connect to device:
Connect to a single device. Optionally if a device requires auth (userAuth flag set to true in device description), user / pass pair needs to be contained in the request body in JSON format. And in this case, a JWT token would be returned in the response body. Client would need to provide this token in request body for subsequent device access.

    POST http://gateway_host_name:3049/device-control/<deviceID>/connect
    (optional) request body:
    { username: <name>,
      password: <pass>
    }
    response: 200 OK / 401 unauthrized

##### Disconnect device:
Disconnect a single device, only successful if device is connected

    POST http://gateway_host_name:3049/device-control/<deviceID>/disconnect
    response: 200 OK / 404 not found / 401 unauthrized

##### Get spec of a single device:
Retrieve the spec of a single device, only successful if device is connected

    GET http://gateway_host_name:3049/device-control/<deviceID>/get-spec
    response: 200 OK / 404 not found / 401 unauthrized
    response body: JSON format of the device spec

##### Device control
Invoke a device control action, only successful if device is connected

    POST http://gateway_host_name:3049/device-control/<deviceID>/invoke-action
    request boy:
    {
      serviceID: <id>,
      actionName: <name>,
      argumentList: {
        <input arg name>: <value>,
        <output arg name>: <value>
    }
    response: 200 OK / 404 not found / 401 unauthrized / 408 timeout / 500 internal error
    response body:
    {
      <output arg1 name>: <value>,
      <output arg2 name>: <value>
    }
    Argument names must conform to the device spec that sent to client

##### Example
Below is a command line example of discover, connect, and read sensor value from TI SensorTag CC2650:

    curl -H "Content-Type: application/json" -X POST http://localhost:3049/discover
    curl -H "Content-Type: application/json" -X GET http://localhost:3049/device-list
    curl -H "Content-Type: application/json" -X POST http://localhost:3049/stop-discover
    curl -H "Content-Type: application/json" -X POST http://localhost:3049/device-control/a540d490-c3ab-4a46-98a9-c4a0f074f4d7/connect
    curl -H "Content-Type: application/json" -X POST -d '{"serviceID":"urn:cdif-net:serviceID:Illuminance","actionName":"getIlluminanceData","argumentList":{"illuminance":0}} ' http://localhost:3049/device-control/a540d490-c3ab-4a46-98a9-c4a0f074f4d7/invoke-action
    curl -H "Content-Type: application/json" -X GET http://localhost:3049/device-control/a540d490-c3ab-4a46-98a9-c4a0f074f4d7/get-spec


Data types and validation
-------------------------
Various kinds of protocols or IoT devices profiles would usually define their own set of data types to communicate and exchange data with devices. For example, Bluetooth LE GATT profile would define 40-bit integer type characteristics, and in ONVIF most of arguments to SOAP calls are complex types with multiple nesting level, mandatory or optional fields in each data object. Since data integrity is vital to security, validations need to be enforced on each device data communication, including action calls and event notifications. However, clients would still hope to have a simple enough representation to describe all different data types that cold be exposed by devices.

The Original UPnP specification has defined a rich set of primitive types for state variables, which we map to characteristics or values in other standards, and also defined keywords such as ```allowedValueRange``` / ```allowedValueList``` to aid data validations. However unfortunately, these are still not sufficient to describe the complex-typed data as defined in other standards. Therefore, to provide a complete solution for data typing and validations would be a real challenge.

Considering these facts, CDIF would try to take following approaches to offer a complete solution for data typing and validations:

* Data would be considered to be either in primitive or complex types
* ```dataType``` keyword inside state variable's definition would be used to describe its type
* CDIF would follow JSON specification and only defines these primitive types: ```boolean```, ```integer```, ```number```, and ```string```
* CDIF's common device model would still utilize ```allowedValueRange``` / ```allowedValueList``` keywords for primitive type data, if any of these keywords are defined.
* Device modules managed by CDIF is responsible for mapping above primitive type data to their native types if needed.
* For complex types, they uniformly takes ```object``` type, and the actual data can be either a JSON ```array``` or ```object```.
* If a state variable is in ```object``` tpye, a ```schema``` keyword must be annotated to the state variable definition. And its value would be used for validation purpose.
* The value of ```schema``` keyword refer to the formal [JSON schema](http://json-schema.org/) definition to this data object. This value is a [JSON pointer](https://tools.ietf.org/html/rfc6901) refers to the variable's schema definition inside device's root schema document, which is either provided by CDIF or device modules. Authenticated clients, such as client web apps or third party web services may also retrieve the schema definitions associated with this reference through CDIF's RESTful interface and do proper validations if needed.
* CDIF would internally resolve the schema definitions associated with this pointer, as either defined by CDIF or its submodules, and do data validations upon action calls or event notifications.

This framework and its [cdif-onvif-manager](https://github.com/out4b/cdif-onvif-manager) implementation contains an example of providing schema definitions, and do data validations to complex-typed arguments to ONVIF camera's PTZ action calls.

Eventing
--------
CDIF implemented a simple [socket.io](socket.io) based server to provide pub / sub model of eventing service. For now CDIF chooses socket.io as the eventing interface because its pub / sub based room API simplified our design. CDIF try not to invent its own pub / sub API but try to follow standardized technologies as much as possible. If there is such requirement in the future, we may extend this interface to support more pub / sub protocols such MQTT, AMQP and etc, given we know how to appropriately apply security means to them.

Event subscriptions in CDIF are service based, which means clients have to subscribe to a specific service ID. If any of the variable state managed by the service are updated, e.g. a sensor value change, or a light bulb is switched on / off, client would receive event updates from CDIF. In addition, CDIF would cache device states upon successful action calls, thus devices doesn't have to send event data packets to be able to notify their state updates. This would improve the usage model of eventing feature, but also leads to a result that, the ```sendEvents``` property of a state variable in CDIF's common device model would have no significance, because in theory, all state variables can be evented if they can be written by any connected client. However, CDIF would still respect this property, and if it is set to false by the device model, clients are not able to receive any event updates from it.

Users may refer to [test/socket.html](https://github.com/out4b/cdif/blob/master/test/socket.html) for a very simple use case on CDIF's eventing interface.

Device presentation
-------------------
Some kinds of IoT devices, such as IP cameras, may have their own device presentation URL for configuration and management purpose. To support this kind of usage, CDIF implemented a reverse proxy server to help redirect HTTP traffics to this URL. By doing this, the actual device presentation URL would be hidden from external network to help improve security. If the device has a presentation URL, its device model spec would have "devicePresentation" flag set to true. After the device successfully connected through CDIF's connect API, its presentation URL is mounted on CDIF's RESTful interface and can be uniformly accessed from below URL:

    http://gateway_host_name:3049/device-control/<deviceID>/presentation/

For now only ONVIF devices support this kind of usage. But this concept should be extensible to any device or manufacturer modules who want to host their own presentation page, given they implemented the internal getDeviceRootUrl() interface which returns the reverse proxy server's root URL. Please refer to [cdif-onvif-manager](https://github.com/out4b/cdif-onvif-manager) module for more information.

Notes
-----
Due to the dependencies to native bindings of the underlying network stacks, CDIF now only support node v0.10.x. Currently it is only tested on Ubuntu 14.x system. If you encountered a problem on Mac or other system, please kindly report the issue [here](https://github.com/out4b/cdif/issues).

Test
----
Open a console and run below command:
```sh
    cd cdif
    npm test
```
The above command will discover, connect all available devices, and then invoke *every* action exposed by its device model spec.

### Acknowlegement
Many thanks to the work contributed by following repositories that made this framework implementation possible:

* [noble-device](https://github.com/sandeepmistry/noble-device), [yeelight-blue](https://github.com/sandeepmistry/node-yeelight-blue), and [node-sensortag](https://github.com/sandeepmistry/node-sensortag)
* [onvif](https://github.com/agsh/onvif)
* [openzwave-shared](https://www.npmjs.com/package/openzwave-shared)
