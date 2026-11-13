# IoT Gateway Cloud Bridge

A Java-based Gateway Device Application (GDA) that bridges IoT edge devices 
to the cloud — handling message brokering, data aggregation, and multi-protocol 
communication.

It is the gateway half of a two-part system. The edge half, the Python Constrained Device Application (CDA), is
[iot-edge-device-simulation](https://github.com/kchimbodza/iot-edge-device-simulation).
Edge device → gateway → cloud: the CDA publishes sensor data over MQTT, this GDA checks it, sends commands back,
and forwards everything to the Ubidots cloud.

## At a glance

| | |
|---|---|
| **What it does** | Receives sensor and CPU/memory data from the edge device, turns the humidifier on or off when humidity stays out of range, and forwards the data to a Ubidots dashboard. A switch on the dashboard turns the device's LED matrix on or off. |
| **Protocols** | MQTT over TLS (local broker and cloud), CoAP server |
| **Tech** | Java 11, Maven, Eclipse Paho MQTT, Eclipse Californium (CoAP), Gson, JUnit 4 |
| **Tests** | 89 JUnit tests: 43 unit, 46 integration |

## Architecture

<p align="center"><img src="docs/architecture.svg" width="100%" alt="Full edge-to-cloud system with this repo, the gateway (GDA, Java), highlighted. Telemetry path: a Sense HAT emulator on the edge device (CDA, Python) produces temperature, humidity, pressure and pitch readings; sensor and actuator managers poll them; the CDA DeviceDataManager publishes JSON over MQTT with TLS to a local broker on port 8883; the GDA DeviceDataManager receives it and checks the humidity rule (outside 30 to 50 percent for 10 seconds sends a humidifier command); the CloudClientConnector, with its own Paho MQTT client, publishes to Ubidots over TLS. Commands go back the other way: humidifier commands and the Ubidots LED switch travel from the GDA through the broker to the CDA. The GDA also runs a Californium CoAP server with telemetry, system-performance and actuator resources, tested on its own."></p>

1. **In from the edge:** `MqttClientConnector` subscribes to the CDA's sensor, actuator-response, system-performance and LED-position topics on a local broker (TLS, port 8883).
2. **Decide:** `DeviceDataManager` checks humidity. If it stays below 30% or above 50% for 10 s, it publishes a humidifier command (target 40%) to the CDA, and turns it off again when humidity is back to 40%.
3. **Out to the cloud:** `CloudClientConnector` publishes each value to Ubidots (`/v1.6/devices/...`) over MQTT with TLS, and listens for the dashboard's LED switch, which it passes down to the CDA.
4. **Also built:** a CoAP server (`CoapServerGateway`) with telemetry, system-performance and actuator-command resources, and a system monitor for the gateway's own CPU, memory and disk use.

**Why this architecture**

| Choice | Why |
|---|---|
| Gateway between device and cloud | Only the gateway holds the cloud credentials and certificate; the edge device only talks to a local broker. |
| MQTT through a broker | The CDA and GDA never need each other's address; each topic (from `ResourceNameEnum`) is one kind of message. |
| TLS on port 8883 with a CA certificate | Sensor data and commands are encrypted on the wire; checked with Wireshark (commit `005524b`). |
| A second MQTT client for the cloud | The local broker and Ubidots have different hosts, credentials and certificates, so each gets its own connection. |
| Humidity rule waits 10 s | One out-of-range reading does not switch the humidifier; the value has to stay out of range. |
| Every part switched on or off in `PiotConfig.props` | MQTT, CoAP, cloud and monitoring can each be run and tested alone. |

## Project Structure

| Module | Description |
|--------|-------------|
| MQTT Client | Subscribes to edge-device topics on the local MQTT broker and sends commands back |
| CoAP Server | Handles constrained device protocol requests |
| Cloud Connector | Forwards aggregated data to cloud services (Ubidots) |
| TLS/SSL | Encrypted end-to-end communication |
| Data Aggregation | Processes and filters incoming sensor streams |

## Results

There is no CI for this repo, and I could not re-run the tests for this README (no Maven in the environment used).
What the repo and its history show:

| Lab module | What was built | Related tests / record in the commit history |
|---|---|---|
| 05 | System performance manager (CPU, memory, disk) and data containers | `SystemPerformanceManagerTest`, data class unit tests |
| 07 | MQTT client in the gateway | MQTT connector and control-packet integration tests |
| 08 | CoAP server with 3 resource handlers | "GDA CoAP Server fully tested with live manual testing - ALL TESTS PASSING" (`6bb138a`) |
| 10 | Humidity rule → humidifier command; TLS to the broker | `DeviceDataManagerSimpleCdaActuationTest`; "TLS connectivity verified with Wireshark" (`005524b`) |
| 11 | Cloud integration with Ubidots | `CloudClientConnectorTest` |
| 12 | LED position from the CDA shown on Ubidots; LED switch from Ubidots to the CDA | Marked "Lab12 Complete" (`0260a39`) |

Tests: 43 unit tests (config, data classes, JSON conversion, system tasks) need no broker. The 46 integration tests
need a running MQTT broker, and some need the CDA or a Ubidots account.

## What worked and what didn't

| | What happened | Notes |
|---|---|---|
| ✅ | Full loop from the edge to the cloud and back | Sensor data goes up to Ubidots; the humidity rule and the dashboard LED switch both send commands down to the CDA (labs 10–12). |
| ✅ | TLS on both MQTT links | Local broker and Ubidots both use port 8883 with a certificate. |
| ❌ | Commands go over MQTT only | Sending commands over CoAP is still a `TODO` in `sendActuatorCommandtoCda`. |
| ❌ | Not finished: MQTT v5 client, Redis and InfluxDB storage, email alerts | `Mqttv5ClientConnector`, `RedisPersistenceAdapter`, `InfluxPersistenceAdapter` and `SmtpClientConnector` are still stubs that return `false`. |
| ❌ | LED vertical position is guessed | The gateway sends Y as `7 - X` to Ubidots instead of reading the Y value the CDA sends. |
| ⚠️ | Hard to run on another machine | Certificate and credential paths in `config/PiotConfig.props` are absolute paths on my laptop, and no tests run in CI. |

## Technologies Used
`Java` `MQTT` `CoAP` `TLS/SSL` `Wireshark`

## Skills Demonstrated
- Gateway device architecture and implementation
- Multi-protocol IoT communication (MQTT + CoAP)
- Encrypted data transmission with certificate authentication
- Cloud integration and data aggregation pipelines

<details>
<summary><b>Run it yourself</b></summary>

Needs Java 11+, Maven, an MQTT broker with TLS on port 8883, and (for the cloud part) a Ubidots account.
Edit `config/PiotConfig.props` first: broker host, certificate paths, credential files and which parts to enable.

```bash
mvn test -Dtest='**/unit/**/*Test'   # unit tests only, no broker needed
mvn package                         # build
```

Then start `programmingtheiot.gda.app.GatewayDeviceApp` (VS Code and Eclipse launch configs are included), and the
[CDA](https://github.com/kchimbodza/iot-edge-device-simulation) on the device side.

</details>

## Built with

Built for a Connected Devices course, starting from the course's
[Programming the IoT](https://github.com/programming-the-iot) Java template by Andrew D. King (MIT licence, see `LICENSE`),
which supplies the interfaces, config utilities and test shells. I implemented the gateway logic in lab modules 5–12.

## Author

**Kudzayi Chimbodza** · [GitHub @kchimbodza](https://github.com/kchimbodza)
