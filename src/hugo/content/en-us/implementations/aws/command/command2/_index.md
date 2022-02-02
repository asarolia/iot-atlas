---
title: "Command - Device Shadow"
weight: 10
summary: "Command and control of a device by using the device shadow leveraging MQTT topics"
---

Command and control is the operation of sending a message to a device requesting it to perform some action or to control the device configuration. However, when interaction happens with devices over intermittent networks, deployed in far-flung areas, or when using devices with limited resources, an IoT solution may not reliably send or receive commands or configuration updates. IoT applications or services can simulate and control edge device configuration and execute actions by using device shadow.

{{% notice note %}}
This implementation is designed using the [AWS IoT device shadow service](https://docs.aws.amazon.com/iot/latest/developerguide/iot-device-shadows.html) through reserved MQTT shadow topics, and describes two approaches that use device shadows. Please refer to the [Designing MQTT Topic for AWS IoT Core](https://docs.aws.amazon.com/whitepapers/latest/designing-mqtt-topics-aws-iot-core/designing-mqtt-topics-aws-iot-core.html), specifically the _Applications on AWS_ section. This white paper provides additional alternative patterns that go beyond the scope of this implementation.
{{% /notice %}}

## Use Cases

- Request or update device state configuration
  - _I want to track and control smart home devices from my mobile application_
  - _I want to remotely control household heaters by turning them on/off or by setting up the desired temperature_ 
- Request device actions based on some commands
  - _I want to unlock a door for a family visitor remotely_ 
  - _I want to remotely instruct a device for some action, the basis of a command_

## Reference Architecture

This section describes architecture considerations for command and control using the AWS IoT Core device shadow
 - When requesting an update to device state configuration
 - When requesting device actions based on commands  

{{< tabs groupId="MQTT-arch" >}}
{{% tab name="Device Configuration" %}}

# ![Track and control device state configuration using shadow ](architecture1.svg)


- _AWS IoT Core_ is the MQTT message broker processing messages
- _Device_ is the IoT thing to be controlled
- _Client Application_ is the remote logic that issues commands, updates device state configuration, and consumes device telemetry data
- _Device Shadow_ is the replica that makes a device state available to applications and other services
- _ShadowTopicPrefix_ is an abbreviated form of the topic, referred to as `$aws/things/device1/shadow`(classic shadow topic)

1. The _Device_ establishes an MQTT connection to the _AWS IoT Core_ endpoint and then subscribes to the reserved [MQTT shadow topics](https://docs.aws.amazon.com/iot/latest/developerguide/reserved-topics.html#reserved-topics-shadow) to get and update shadow event operations. Moreover, instead of subscribing individually to each reserved shadow topic, a wildcard(_#_) could be used for a generalized subscription-like _`ShadowTopicPrefix/get/#`_ , _`ShadowTopicPrefix/update/#`_.
2. After successfully connecting and topic subscription, the _Device_ publish request to _`ShadowTopicPrefix/get`_ topic and process the latest shadow document received on the _`ShadowTopicPrefix/get/accepted`_ topic. After processing configuration changes from _desired_ data in the shadow document, _Device_ makes publish request to _`ShadowTopicPrefix/update`_ topic with device latest configuration as _reported_ data on shadow document.     
3. After processing, delta attributes on initial connect/reconnect. If the device optionally remains connected, it can further receive delta changes on the shadow document from the _Client Application_.   
4. The _Client Application_ establishes an MQTT connection to the _AWS IoT Core_ endpoint and then subscribes to the _`ShadowTopicPrefix/update/accepted`_, _`ShadowTopicPrefix/update/rejected`_ shadow topics to process configuration updates done on the device end. 
5. For updating the device configuration, _Client Application_ publishes a message as _desired_ state on _`ShadowTopicPrefix/update`_ reserved topic. 

```plantuml
@startuml
!define AWSPuml https://raw.githubusercontent.com/awslabs/aws-icons-for-plantuml/v7.0/dist
!includeurl AWSPuml/AWSCommon.puml
!includeurl AWSPuml/InternetOfThings/all.puml
!includeurl AWSPuml/General/Client.puml

'Comment out to use default PlantUML sequence formatting
skinparam participant {
    BackgroundColor AWS_BG_COLOR
    BorderColor AWS_BORDER_COLOR
}
'Hide the bottom boxes
hide footbox

participant "<$IoTGeneric>\nDevice" as device
participant "<$IoTShadow>\nShadow" as shadow
participant "<$IoTCore>\nMQTT Broker" as broker
participant "<$Client>\nApplication" as app

== Device connect and subscribe ==
device -> broker : connect(iot_endpoint)
device -> broker : subscribe("$aws/things/device1/shadow/get/accepted")
device -> broker : subscribe("$aws/things/device1/shadow/get/rejected")
device -> broker : subscribe("$aws/things/device1/shadow/update/accepted")
device -> broker : subscribe("$aws/things/device1/shadow/update/rejected")
device -> broker : subscribe("$aws/things/device1/shadow/update/delta")

== Application connect and subscribe ==
app -> broker : connect(iot_endpoint)
app -> broker : subscribe("$aws/things/device1/shadow/update/accepted")
app -> broker : subscribe("$aws/things/device1/shadow/update/rejected")

== App to Device state control and update ==
app -> broker : publish("$aws/things/device1/shadow/update", "desired : {motorStatus: on}")
broker -> shadow : updates shadow for desired state changes from app
device -> broker : On initial connect/reconnect, publish("$aws/things/device1/shadow/get")
broker -> device : Send persisted shadow state, publish("$aws/things/device1/shadow/get/accepted") 
device -> device : Turn motor on
device -> broker : publish("$aws/things/device1/shadow/update", "reported : {motorStatus: on}")
broker -> shadow : updates reported state on shadow document
broker -> app : publish("$aws/things/device1/shadow/update/accepted", "success")
app -> app : reconcile("device1")
rnote over device #FFFFFF: If device remained connected
app -> broker : publish("$aws/things/device1/shadow/update", "desired : {motorStatus: off}")
broker -> shadow : updates shadow for desired state changes from app
broker -> device : publish("$aws/things/device1/shadow/update/delta", "desired : {motorStatus: off}")
device -> device : Turn motor off
device -> broker : publish("$aws/things/device1/shadow/update", "reported : {motorStatus: off}")
broker -> shadow : updates reported state on shadow document
broker -> app : publish("$aws/things/device1/shadow/update/accepted", "success")
app -> app : reconcile("device1")


@enduml
```



{{% /tab %}}
{{% tab name="Device Action" %}}


![Command and control device action using shadow](architecture2.svg)

- _AWS IoT Core_ is the MQTT message broker processing messages
- _Device_ is the IoT thing to be controlled
- _Client Application_ is the remote logic that issues commands, updates device configuration, and consumes device telemetry data
- _Device Shadow_ is the replica that makes device state available to applications and other services
- _ShadowTopicPrefix_ is an abbreviated form of the topic, referred to as `$aws/things/device1/shadow` (classic shadow)

1. The _Device_ establishes an MQTT connection to the _AWS IoT Core_ endpoint and then subscribes to the reserved [MQTT shadow topics](https://docs.aws.amazon.com/iot/latest/developerguide/reserved-topics.html#reserved-topics-shadow) to get and update shadow event operations. Moreover, instead of subscribing individually to each reserved shadow topic, a wildcard(_#_) could be used for a generalized subscription-like _`ShadowTopicPrefix/get/#`_ , _`ShadowTopicPrefix/update/#`_.
2. After successfully connecting and topic subscription, the _Device_ publishes a request to _`ShadowTopicPrefix/get`_ topic and process the latest shadow document received on the _`ShadowTopicPrefix/get/accepted`_ topic.
3. After receiving the _command_ on shadow document,  the _Device_ makes publish request to `cmd/device1/response` topic as a command execution response. On completion of a command,  the _Device_  publish status on `ShadowTopicPrefix/update` topic for reconciliation of _reported_ state on shadow document.     
4. After processing, delta attributes on initial connect/reconnect. If the device optionally remains connected, it can further receive delta changes on the shadow document from the _Client Application_.  
5. The _Client Application_ establishes an MQTT connection to the _AWS IoT Core_ endpoint and then subscribes to the `ShadowTopicPrefix/update/accepted`,  `ShadowTopicPrefix/update/rejected` shadow topics to reconcile shadow document updates made by the device. The _Client Application_ also subscribes to the `cmd/device1/response` topic to process command execution response.
6. To send a command, the _Client Application_ publishes a message on the `ShadowTopicPrefix/update` topic by adding a command request under _desired_ state data for the device.



```plantuml
@startuml
!define AWSPuml https://raw.githubusercontent.com/awslabs/aws-icons-for-plantuml/v7.0/dist
!includeurl AWSPuml/AWSCommon.puml
!includeurl AWSPuml/InternetOfThings/all.puml
!includeurl AWSPuml/General/Client.puml

'Comment out to use default PlantUML sequence formatting
skinparam participant {
    BackgroundColor AWS_BG_COLOR
    BorderColor AWS_BORDER_COLOR
}
'Hide the bottom boxes
hide footbox

participant "<$IoTGeneric>\nDevice" as device
participant "<$IoTShadow>\nShadow" as shadow
participant "<$IoTCore>\nMQTT Broker" as broker
participant "<$Client>\nApplication" as app

== Device connect and subscribe ==
device -> broker : connect(iot_endpoint)
device -> broker : subscribe("$aws/things/device1/shadow/get/accepted")
device -> broker : subscribe("$aws/things/device1/shadow/get/rejected")
device -> broker : subscribe("$aws/things/device1/shadow/update/accepted")
device -> broker : subscribe("$aws/things/device1/shadow/update/rejected")
device -> broker : subscribe("$aws/things/device1/shadow/update/delta")

== Application connect and subscribe ==
app -> broker : connect(iot_endpoint)
app -> broker : subscribe("$aws/things/device1/shadow/update/accepted")
app -> broker : subscribe("$aws/things/device1/shadow/update/rejected")
app -> broker : subscribe("cmd/device1/response")

== App to Device command control and response ==
app -> broker : publish("$aws/things/device1/shadow/update", "desired : {"command": [{ "do-something":{"payload":"goes here"} }]}")
broker -> shadow : updates shadow for desired command message from app
device -> broker : on initial connect/reconnect, publish("$aws/things/device1/shadow/get")
broker -> device : send persisted shadow state, publish("$aws/things/device1/shadow/get/accepted") 
device -> device : do-something as per command
device -> broker : On completion of command update reported state, publish("$aws/things/device1/shadow/update", "reported : {"command": [{ "do-something":{"payload":"goes here"} }]}")
broker -> shadow : updates reported state on shadow document
broker -> app : publish("$aws/things/device1/shadow/update/accepted", "success")
app -> app : reconcile("device1")
rnote over device #FFFFFF: If device remained connected
app -> broker : publish("$aws/things/device1/shadow/update", "desired : {"command": [{ "do-something":{"payload":"updated value"} }]}")
broker -> shadow : updates shadow for desired command message from app
broker -> device : publish("$aws/things/device1/shadow/update/delta", "desired : {"command": [{ "do-something":{"payload":"updated value"} }]}"")
device -> device : do-something as per command
device -> broker : On completion of command update reported state, publish("$aws/things/device1/shadow/update", "reported : {"command": [{ "do-something":{"payload":"updated value"} }]}")
broker -> shadow : updates reported state on shadow document
broker -> app : publish("$aws/things/device1/shadow/update/accepted", "success")
app -> app : reconcile("device1")

@enduml
```
{{% /tab %}}
{{< /tabs >}}

### Assumptions

- Use of classic shadow instead of a named shadow.
- The _Device_ shadow is pre-created before the device performs the get request on connect/re-connect.
- The _Device_ remains connected to the _AWS IoT Core_ endpoint after the initial connection; the device waits for the updates to the desired configuration or command.
- All participants successfully receive, forward, and act upon the message flows in either direction.


## Implementation

Both the _Client Application_ and the _Device_ can use a similar approach while connecting, sending, and receiving messages. Therefore, code samples below are complete for each participant.

{{% notice note %}}
The code samples focus on the _command_ design in general. Please refer to the [Getting started with AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/iot-gs.html) for details on creating things, certificates, and obtaining your endpoint. The code samples demonstrate the essential capability of the _Command_ pattern.
{{% /notice %}}

### Device

The _Device_ code focuses on connecting to the _Broker_ and then subscribing to reserved shadow topics to receive commands and configuration changes. Upon receiving a command, the _Device_ performs some local action and then publishes the outcome (success, failure) of the command back to the _Client Application_, which then reconciles the command. After that, the _Device_ will continue to receive and respond to configuration updates or commands or both until stopped.

Please refer to this [shadow sample](https://github.com/aws/aws-iot-device-sdk-python-v2/blob/main/samples/shadow.py) for more details.

- Install SDK from PyPI: `python3 -m pip install awsiotsdk`
- Replace the global variables with valid endpoint, clientId, and credentials
- Start in a separate terminal session before running the _Application_: `python3 client.py`

```python
# client.py - This sample demonstrates waiting for command or configuration updates for evaluation & processing.

from awscrt import auth, io, mqtt, http
from awsiot import iotshadow
from awsiot import mqtt_connection_builder
from concurrent.futures import Future
import sys
import json
import datetime as datetime
import time as time

###################################################################################################################
# This sample uses the AWS IoT Device Shadow Service to receive commands and keep a property in sync between device and server. At start-up, the device requests the shadow document to learn the property's initial state and process the attributes and commands passed in the desired value of the shadow document. The device also subscribes to "delta" events from the AWS IoT core endpoint, which get published when a property's "desired" value differs from its "reported" value. When the device learns of a new desired value, that data is processed, and a new "reported" value is pushed to the server. In this sample implementation, an IoT edge device acts as an actuator to switch on/off the motor via the shadow document attribute. Additionally, the device will act on the command passed from the client application on the device shadow document.
###################################################################################################################

# Using globals to simplify sample code

client_id = "REPLACE_WITH_DEVICE_ID"
client_token = "REPLACE_WITH_SECURE_DEVICE_TOKEN"
endpoint = "REPLACE_WITH_YOUR_ENDPOINT_FQDN"
client_certificate = "PATH_TO_CLIENT_CERTIFICATE_FILE"
client_private_key = "PATH_TO_CLIENT_PRIVATE_KEY_FILE"
root_ca = "PATH_TO_ROOT_CA_CERTIFICATE_FILE"
thing_name = "REPLACE_WITH_DEVICE_NAME"

# Dummy state variables to showcase sample program logic and validation. 
# For implementation during device connect or reconnect, read this data as a last processed state from the device store or via the last reported state in the shadow document.

DEVICE_MOTOR_STATUS = "off"
SHADOW_DOCUMENT_VERSION = 0

# For simplicity, this implementation uses single-threaded operation. In the case of multi-threaded use cases, consider utilizing a lock. More details here - https://github.com/aws/aws-iot-device-sdk-python-v2/blob/main/samples/shadow.py

# Function for gracefully quitting this sample
def exit(msg_or_exception):
    
    print("Exiting sample:", msg_or_exception)
    future = mqtt_connection.disconnect()
    future.add_done_callback(on_disconnected)
    
def on_disconnected(disconnect_future):
    print("Disconnected.")

# Call-back function to encode a payload into JSON
def json_encode(string):
        return json.dumps(string)

# Call-back to publish message on specific topic
def send(topicname,message):
    mqtt_connection.publish(
                topic=topicname,
                payload=message,
                qos=mqtt.QoS.AT_LEAST_ONCE)
    print("Message published on topic - '{}'.".format(topicname))

def on_get_shadow_accepted(response):
    # type: (iotshadow.GetShadowResponse) -> None
    global SHADOW_DOCUMENT_VERSION
    global DEVICE_MOTOR_STATUS
    global thing_name
    global client_token
    processedAttrib = False
    processedCommand = False
    print ("Processing response on GetShadow")
  
    try:
        
        if response is not None:
            if response.client_token != client_token:
                print("Invalid client token passed")
                return
        
        if response.state is not None:
            if response.version is not None and response.version <= SHADOW_DOCUMENT_VERSION:
                print("No new data to process, skipping get shadow document processing!")
                return
            
            # Check if any delta changes to process    
            if response.state.delta is not None:
                
                # Update latest shadow document version
                SHADOW_DOCUMENT_VERSION = response.version
                #########################################################################
                # Code to push processed shadow document version to device storage goes here
                # store__to_local_storage(SHADOW_DOCUMENT_VERSION)
                # We have just updated global variable to simplify code
                #########################################################################
                print("Updated local shadow document version on connect or reconnect")
                
                # Check for any device attribute/configuration changes to process 
                if response.state.delta.get("deviceAttrib") is not None:
                    DEVICE_MOTOR_STATUS = response.state.delta.get("deviceAttrib").get("motorStatus")
                    print("Motor is turned - {}".format(DEVICE_MOTOR_STATUS))
                    processedAttrib = True
                        
                # Check if any command passed for device action
                if response.state.delta.get("command") is not None:
                    
                    #Loop through all the commands for processing 
                    for eachcmd in response.state.delta.get("command"):
                        
                        ####################################################
                        # Command processing code goes here
                        ####################################################
                        if "relay-command" in eachcmd:
                            if eachcmd.get("relay-command").get("ttl") is not None:
                                value_ttl = eachcmd.get("relay-command").get("ttl")
                            
                            # Bypass command execution if lag between system time and time stamp  
                            # on shadow document is beyond command threshold TTL
                            if (int(datetime.datetime.now().timestamp()) - int(response.timestamp.timestamp()) > value_ttl):
                                print("Command validity expired, skipping action")
                                continue
                            else:
                                # Action to perform on device for completing the command
                                
                                print("Executing command action..")
                                
                                # Actual code for command action goes here. For this sample we 
                                # are just relaying the message on specific topic    
                                value_message = eachcmd.get("relay-command").get("message")
                                relay_message = {'msg':value_message,'deviceId': thing_name}
                                value_topic = eachcmd.get("relay-command").get("topic")
                                send(value_topic,json_encode(relay_message))
                                processedCommand = True

                #Payload for Reported state on shadow document
                if processedAttrib and processedCommand:
                    shadowReportedMessage = {"deviceAttrib":{"motorStatus":DEVICE_MOTOR_STATUS},"command":[{"relay-command":{"message":value_message,"topic":value_topic,"ttl":value_ttl}}]}
                elif processedAttrib and not processedCommand:
                    shadowReportedMessage = {"deviceAttrib":{"motorStatus":DEVICE_MOTOR_STATUS}}
                elif processedCommand and not processedAttrib:
                    shadowReportedMessage = {"command":[{"relay-command":{"message":value_message,"topic":value_topic,"ttl":value_ttl}}]}
                    
                if processedAttrib or processedCommand:
                    
                    # Update reported state on device shadow
                    # use a unique token so we can correlate this "request" message to
                    # any "response" messages received on the /accepted and /rejected topics
                    token = client_token
                    request = iotshadow.UpdateShadowRequest(
                                thing_name=thing_name,
                                state=iotshadow.ShadowState(
                                    reported = shadowReportedMessage
                                ),
                                client_token=token,
                            )
                
                    future = shadow_client.publish_update_shadow(request, mqtt.QoS.AT_LEAST_ONCE)
                    future.add_done_callback(on_publish_update_shadow)
                    print ("Device shadow updated during get accepted")
                    return
          
        print("Get response has no state for processing")
        return

    except Exception as e:
        exit(e)

def on_get_shadow_rejected(error):
    # type: (iotshadow.ErrorResponse) -> None
    try:
        
        if error.code == 404:
            print("Thing has no shadow document. Consider creating shadow first")
        else:
            exit("Get request was rejected. code:{} message:'{}'".format(
                error.code, error.message))

    except Exception as e:
        exit(e)

def on_shadow_delta_updated(delta):
    # type: (iotshadow.ShadowDeltaUpdatedEvent) -> None
    global SHADOW_DOCUMENT_VERSION
    global DEVICE_MOTOR_STATUS
    global thing_name
    global client_token
    processedAttrib = False
    processedCommand = False
    print("Processing delta response from device shadow")
    
    try:
        if delta.state is not None:
            if delta.version is not None and delta.version <= SHADOW_DOCUMENT_VERSION:
                print("No new data to process, skipping delta shadow document processing!")
                return
            
            # Update latest shadow document version
            SHADOW_DOCUMENT_VERSION = delta.version
            ##################################################################################
            # Code to push currently processed shadow document version to device storage goes here
            # store__to_local_storage(SHADOW_DOCUMENT_VERSION)
            # We have just updated global variable to simplify code
            ##################################################################################
            print("Updated local shadow document version for delta updates")
            
            # Check for any device attribute/configuration changes to process 
            if delta.state.get("deviceAttrib") is not None:
                DEVICE_MOTOR_STATUS = delta.state.get("deviceAttrib").get("motorStatus")
                print("Motor is turned - {}".format(DEVICE_MOTOR_STATUS))
                processedAttrib = True
                    
            # Check if any command passed for device action.
            # Command array enables processing of multiple commands
            if delta.state.get("command") is not None:
                
                #Loop through all the commands for processing 
                for eachcmd in delta.state.get("command"):
                    
                    ###########################################################
                    # Command processing code goes here.
                    # we are just relaying message back on topic for simplicity
                    ###########################################################
                    if "relay-command" in eachcmd:
                        if eachcmd.get("relay-command").get("ttl") is not None:
                            value_ttl = eachcmd.get("relay-command").get("ttl")
                        
                        # Bypass command execution if lag between system time and 
                        # time stamp on shadow document is beyond command threshold TTL
                        if (int(datetime.datetime.now().timestamp()) - int(delta.timestamp.timestamp()) > value_ttl):
                            print("Command validity expired, skipping action")
                            continue
                        else:
                            # Action to perform on device for completing the command
                            
                            print("Executing command action..")
                            
                            value_message = eachcmd.get("relay-command").get("message")
                            relay_message = {'msg':value_message,'deviceId': thing_name}
                            value_topic = eachcmd.get("relay-command").get("topic")
                            send(value_topic,json_encode(relay_message))
                            processedCommand = True
    
            #Payload for Reported state on shadow document
            if processedAttrib and processedCommand:
                shadowReportedMessage = {"deviceAttrib":{"motorStatus":DEVICE_MOTOR_STATUS},"command":[{"relay-command":{"message":value_message,"topic":value_topic,"ttl":value_ttl}}]}
            elif processedAttrib and not processedCommand:
                shadowReportedMessage = {"deviceAttrib":{"motorStatus":DEVICE_MOTOR_STATUS}}
            elif processedCommand and not processedAttrib:
                shadowReportedMessage = {"command":[{"relay-command":{"message":value_message,"topic":value_topic,"ttl":value_ttl}}]}
                
            if processedAttrib or processedCommand:
                
                ## Update device shadow reported state
                # use a unique token so we can correlate this "request" message to
                # any "response" messages received on the /accepted and /rejected topics
                token = client_token
                request = iotshadow.UpdateShadowRequest(
                            thing_name=thing_name,
                            state=iotshadow.ShadowState(
                                reported = shadowReportedMessage
                            ),
                            client_token=token,
                        )
            
                future = shadow_client.publish_update_shadow(request, mqtt.QoS.AT_LEAST_ONCE)
                future.add_done_callback(on_publish_update_shadow)
                print ("Device shadow updated during get accepted")
                return
         
        else:
            print("Delta response has no state changes")
            return
            
    except Exception as e:
        exit(e)

def on_publish_update_shadow(future):
    #type: (Future) -> None
    try:
        future.result()
        print("Update request published.")
    except Exception as e:
        print("Failed to publish update request.")
        exit(e)

def on_update_shadow_accepted(response):
    # type: (iotshadow.UpdateShadowResponse) -> None
    try:
       
       print("Shadow update acepted - {}".format(response))

    except Exception as e:
        exit(e)

def on_update_shadow_rejected(error):
    # type: (iotshadow.ErrorResponse) -> None
    try:
       
        exit("Update request was rejected. code:{} message:'{}'".format(
            error.code, error.message))

    except Exception as e:
        exit(e)


if __name__ == '__main__':
    
    io.init_logging(getattr(io.LogLevel, "Info"), "stderr")


    # Create SDK based resources
    event_loop_group = io.EventLoopGroup(1)
    host_resolver = io.DefaultHostResolver(event_loop_group)
    client_bootstrap = io.ClientBootstrap(event_loop_group, host_resolver)

    # Create native MQTT connection from credentials on path (filesystem)
    mqtt_connection = mqtt_connection_builder.mtls_from_path(
        endpoint=endpoint,
        cert_filepath=client_certificate,
        pri_key_filepath=client_private_key,
        client_bootstrap=client_bootstrap,
        ca_filepath=root_ca,
        client_id=client_id,
        clean_session=True,
        keep_alive_secs=30,
    )

    print("Connecting to {} with client ID '{}'...".format(
        endpoint, client_id))

    connected_future = mqtt_connection.connect()

    shadow_client = iotshadow.IotShadowClient(mqtt_connection)

    # Wait for connection to be fully established.
    # Note that it's not necessary to wait, commands issued to the
    # mqtt_connection before its fully connected will simply be queued.
    # But this sample waits here so it's obvious when a connection
    # fails or succeeds.
    connected_future.result()
    print("Connected!")

    try:
        # Subscribe to necessary topics.
        # Note that is **is** important to wait for "accepted/rejected" subscriptions
        # to succeed before publishing the corresponding "request".
        print("Subscribing to Update responses...")
        update_accepted_subscribed_future, _ = shadow_client.subscribe_to_update_shadow_accepted(
            request=iotshadow.UpdateShadowSubscriptionRequest(thing_name=thing_name),
            qos=mqtt.QoS.AT_LEAST_ONCE,
            callback=on_update_shadow_accepted)

        update_rejected_subscribed_future, _ = shadow_client.subscribe_to_update_shadow_rejected(
            request=iotshadow.UpdateShadowSubscriptionRequest(thing_name=thing_name),
            qos=mqtt.QoS.AT_LEAST_ONCE,
            callback=on_update_shadow_rejected)

        # Wait for subscriptions to succeed
        update_accepted_subscribed_future.result()
        update_rejected_subscribed_future.result()

        print("Subscribing to Get responses...")
        get_accepted_subscribed_future, _ = shadow_client.subscribe_to_get_shadow_accepted(
            request=iotshadow.GetShadowSubscriptionRequest(thing_name=thing_name),
            qos=mqtt.QoS.AT_LEAST_ONCE,
            callback=on_get_shadow_accepted)

        get_rejected_subscribed_future, _ = shadow_client.subscribe_to_get_shadow_rejected(
            request=iotshadow.GetShadowSubscriptionRequest(thing_name=thing_name),
            qos=mqtt.QoS.AT_LEAST_ONCE,
            callback=on_get_shadow_rejected)

        # Wait for subscriptions to succeed
        get_accepted_subscribed_future.result()
        get_rejected_subscribed_future.result()

        print("Subscribing to Delta events...")
        delta_subscribed_future, _ = shadow_client.subscribe_to_shadow_delta_updated_events(
            request=iotshadow.ShadowDeltaUpdatedSubscriptionRequest(thing_name=thing_name),
            qos=mqtt.QoS.AT_LEAST_ONCE,
            callback=on_shadow_delta_updated)

        # Wait for subscription to succeed
        delta_subscribed_future.result()


        # Issue request for shadow's current state.
        # The response will be received by the on_get_accepted() callback
        print("Requesting current shadow state...")

        # use a unique token so we can correlate this "request" message to
        # any "response" messages received on the /accepted and /rejected topics
        token = client_token

        publish_get_future = shadow_client.publish_get_shadow(
            request=iotshadow.GetShadowRequest(thing_name=thing_name, client_token=token),
            qos=mqtt.QoS.AT_LEAST_ONCE)
     
        # Ensure that publish succeeds
        publish_get_future.result()

    except Exception as e:
        exit(e)

    # Wait timer to receive delta events when device remain connected after initial connect/reconnect   
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        # Disconnect
        print("Disconnecting...")
        disconnect_future = mqtt_connection.disconnect()
        disconnect_future.result()
        print("Disconnected!")

```
- Sample shadow document schema used for sending update requests from the _Client Application_

```json

{
  "state": {
    "desired": {
      "deviceAttrib" :{
      		"motorStatus" : "on"
      },
      "command":[{
      		"relay-command":{
                "message": "Message sent as part of command action",
      		    "topic":"cmd/device1/response",
      		    "ttl":300
              }
      }]
    },
    "reported": {
      "deviceData" :{
      		"motorStatus" : "off"
      }
    }
  },
  "clientToken":"myapp123"
}

```

### Application

The _Client Application_ code connects to the Broker and subscribes to the _Device_'s response topic and reserved MQTT topics for device shadow. The _Client Application_ can process the reported state using reserved shadow topics. The reference implementation for complete processing on the _Client Application_ can be tweaked utilizing the device implementation above.

The below implementation uses AWS CLI to quickly demonstrate how an application can interact with a shadow using the command line.

- To get the current state of the shadow using the AWS CLI. From the command line, enter this command.

    ```cmd

    aws iot-data get-thing-shadow --thing-name REPLACE_WITH_DEVICE_NAME  /dev/stdout

    ```

- To update the shadow from the command line using AWS CLI

    ```cmd


    aws iot-data update-thing-shadow --thing-name REPACE_WITH_DEVICE_NAME \
        --cli-binary-format raw-in-base64-out \
        --payload '{"state":{"desired":{"deviceAttrib":{"motorStatus":"on"},"command":[{"relay-command":{"message":"Message to relay back","topic":"relay-command/response","ttl":300}}]}},"clientToken":"myapp123"}' /dev/stdout

    ```


## Considerations

This implementation covers the basics of a command-and-control pattern. However, it does not cover certain aspects that may arise in production use.

#### Choosing to use named or unnamed shadows
The Device Shadow service supports named and unnamed classic shadows, as used in the past. A thing object can have multiple named shadows and no more than one classic shadow. In addition, a thing object can have both named and unnamed classic shadows simultaneously; however, the API used to access each is slightly different, so it might be more efficient to decide which type of shadow would work best for your solution and use that type only. For more information about the API to access the shadows, see [Shadow topics](https://docs.aws.amazon.com/iot/latest/developerguide/reserved-topics.html#reserved-topics-shadow).

#### Message Versioning and Order
There is no guarantee that the device will receive the AWS IoT service messages in order. Thus, to avoid stale configuration updates, the device can ignore previous state updates by leveraging the version number within each message and skip processing old versions, see [Shadow service document](https://docs.aws.amazon.com/iot/latest/developerguide/device-shadow-document.html#device-shadow-example-response-json-delta).

#### Detecting Device is Connected
Last Will Testament (LWT) is a feature of MQTT that AWS IoT Core supports. It is a message that the MQTT client configures to be sent to an MQTT topic when the device gets disconnected. LWT can be leveraged to set a property of the device shadow, such as a connected property, and allows other apps and services to know the connected state of the device. See [Detecting a device is connected](https://docs.aws.amazon.com/iot/latest/developerguide/device-shadow-comms-app.html) for more detail.

#### Message Size
Many IoT devices have limited bandwidth available to them, which requires communication to be optimized. One way to mitigate the issue is to trim down MQTT messages and publish them to another topic for consumption by the device. For example, set up a rule to take messages from the shadow/update topic, trim them, and publish to shadow/trimmed/update topic for the device to subscribe. See [Rules for AWS IoT](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html) for more detail.

Another consideration with the size is the AWS IoT Device Shadow document size limit of 8KB. This size limit applies to the overall message document, including the desired and actual state. So, if there are enough changes to the desired and reported state, try shortening field names inside your JSON shadow document or create more named shadows for the device. A device can have unlimited things/shadows associated with it. The only requirement is that each name must be unique in your account.

#### Tracking command progress
Each command should have a unique solution type, and each command message should contain a globally unique message ID. The command message’s ID allows the solution to track the status of distinct commands. In addition, the command type enables the diagnosis of any potential issues across categories of commands over time. Finally, the messages should be idempotent and not be missed or duplicated without knowledge of the device and requestor.

#### Do some commands in the solution run significantly longer than the norm?
A simple SUCCESS or FAILURE command completion message will not suffice when some commands run longer than the norm. Instead, the solution should leverage at least three command states: SUCCESS, FAILURE, and RUNNING. In addition, the device should return RUNNING  on an expected interval until the command completes. By using a RUNNING state reported on a scheduled frequency, a solution can determine when a long-running command fails quietly.

#### Application Integration
This example demonstrated how a device and application could interact with the AWS IoT Device Shadow service. However, the service also supports a REST API which may integrate better with specific applications. For example, a mobile application that interacts with other REST APIs would be a good candidate for using the Device Shadow REST API. See [Device Shadow REST API](https://docs.aws.amazon.com/iot/latest/developerguide/device-shadow-rest-api.html) for more detail.

Also, consider how an application could use AWS IoT Rules. For example, you could leverage rules to push a notification to an application or run some custom logic before delivering the data to the mobile application. Using rules can make integrating with your application easier or bring additional features to your users. See [Rules for AWS IoT](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html) for more detail.