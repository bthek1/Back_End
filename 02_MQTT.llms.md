# MQTT

Setting up **MQTT** on Linux involves configuring a **broker**, optionally securing it, and connecting **clients** (like IoT devices, sensors, or Raspberry Pi apps) to publish/subscribe to topics.

Author

Benedict Thekkel

## 📦 1. What is MQTT?

**MQTT (Message Queuing Telemetry Transport)** is a lightweight publish/subscribe messaging protocol designed for IoT and low-bandwidth systems.

- Broker = server (e.g., Mosquitto)
- Client = device/app that publishes or subscribes to topics
- Topic = a UTF-8 string like `sensor/temp/office`

------------------------------------------------------------------------

## ⚙️ 2. Install an MQTT Broker (Mosquitto) on Linux

### ✅ Ubuntu / Debian

``` bash
sudo apt update
sudo apt install -y mosquitto mosquitto-clients
```

### ✅ Fedora / CentOS / RHEL

``` bash
sudo dnf install -y mosquitto mosquitto-clients
```

### ✅ Arch Linux

``` bash
sudo pacman -S mosquitto mosquitto-clients
```

### Start & Enable:

``` bash
sudo systemctl enable mosquitto
sudo systemctl start mosquitto
```

Check status:

``` bash
sudo systemctl status mosquitto
```

------------------------------------------------------------------------

## 🔎 3. Test MQTT Locally

In one terminal (subscriber):

``` bash
mosquitto_sub -h localhost -t "test/topic"
```

In another (publisher):

``` bash
mosquitto_pub -h localhost -t "test/topic" -m "Hello MQTT!"
```

You should see the message appear in the first terminal.

------------------------------------------------------------------------

## 🔐 4. Add Authentication (Username + Password)

1.  **Create password file**

``` bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd myuser
```

2.  **Edit Mosquitto config:** Create or modify `/etc/mosquitto/conf.d/auth.conf`:

``` ini
allow_anonymous false
password_file /etc/mosquitto/passwd
listener 1883
```

3.  **Restart Mosquitto:**

``` bash
sudo systemctl restart mosquitto
```

4.  **Test authenticated connection:**

``` bash
mosquitto_pub -h localhost -t "secure/test" -u myuser -P your_password -m "Secure test"
```

------------------------------------------------------------------------

## 🔒 5. Enable TLS Encryption (Optional but Recommended)

### Create a self-signed cert:

``` bash
mkdir ~/mosquitto_certs && cd ~/mosquitto_certs
openssl req -x509 -newkey rsa:2048 -days 365 -nodes -keyout mqtt.key -out mqtt.crt
```

### Configure TLS in `/etc/mosquitto/conf.d/tls.conf`:

``` ini
listener 8883
cafile /path/to/mqtt.crt
certfile /path/to/mqtt.crt
keyfile /path/to/mqtt.key
require_certificate false
```

Restart:

``` bash
sudo systemctl restart mosquitto
```

Test:

``` bash
mosquitto_pub --cafile mqtt.crt -h localhost -p 8883 -t "secure/topic" -m "Encrypted"
```

------------------------------------------------------------------------

## 🧠 6. Configure Persistence (Retain Data After Restart)

In `/etc/mosquitto/mosquitto.conf`:

``` ini
persistence true
persistence_location /var/lib/mosquitto/
```

Also useful:

``` ini
log_dest file /var/log/mosquitto/mosquitto.log
```

Restart the broker after changing configs.

------------------------------------------------------------------------

## 📲 7. Connect Devices / Clients

### Python (with `paho-mqtt`):

``` bash
pip install paho-mqtt
```

``` python
import paho.mqtt.client as mqtt

client = mqtt.Client()
client.connect("localhost", 1883, 60)
client.publish("test/topic", "Hello from Python!")
client.loop_start()
```

### Raspberry Pi Pico W (MicroPython):

Use `umqtt.simple` or `umqtt.robust` libraries to connect and publish.

------------------------------------------------------------------------

## 📡 8. Remote Access and Firewalls

- Open port `1883` or `8883` in your firewall/router.
- Use `ufw` if on Ubuntu:

``` bash
sudo ufw allow 1883/tcp
```

------------------------------------------------------------------------

## 🛠 9. Troubleshooting Tips

| Issue | Solution |
|----|----|
| Broker not responding | `systemctl status mosquitto` |
| Can’t connect remotely | Check firewall or bind address |
| Auth fails | Re-check `/etc/mosquitto/passwd` and `allow_anonymous` |
| TLS errors | Check cert paths and validity |

------------------------------------------------------------------------

## 📘 10. Extras & Tools

- 🖥 **MQTT Dashboards**: [MQTT Explorer](https://mqtt-explorer.com/), [Node-RED](https://nodered.org/)
- 🛠 **Cloud MQTT brokers**: [HiveMQ Cloud](https://www.hivemq.com/mqtt-cloud-broker/), [Adafruit IO](https://io.adafruit.com/)
- 📦 **Docker Mosquitto**:

``` bash
docker run -it -p 1883:1883 eclipse-mosquitto
```

------------------------------------------------------------------------

## ✅ Summary Table

| Feature     | Default | Config File                       |
|-------------|---------|-----------------------------------|
| Port        | 1883    | `/etc/mosquitto/mosquitto.conf`   |
| Auth        | Off     | `/etc/mosquitto/conf.d/auth.conf` |
| TLS         | Off     | `/etc/mosquitto/conf.d/tls.conf`  |
| Persistence | Off     | `persistence true`                |
| Logs        | Syslog  | or `log_dest file`                |

------------------------------------------------------------------------

## MQTT through python

``` python
import paho.mqtt.client as mqtt
import uuid

# MQTT broker configuration
BROKER = "localhost"  # Change if needed
PORT = 1883
TOPIC = "#"
CLIENT_ID = f"jupyter-client-{uuid.uuid4()}"

# --- Callback Implementations ---

def on_connect(client, userdata, flags, reason_code, properties):
    print(f"[on_connect] Connected: {reason_code}")
    if hasattr(flags, "session_present"):
        print(f"  -> session_present: {flags.session_present}")
    client.subscribe(TOPIC)

def on_connect_fail(client, userdata):
    print("[on_connect_fail] Connection failed.")

def on_disconnect(client, userdata, reason_code, properties):
    print(f"[on_disconnect] Disconnected: {reason_code}")

def on_message(client, userdata, msg):
    print(f"[on_message] Topic: {msg.topic} | Payload: {msg.payload.decode()}")

def on_publish(client, userdata, mid, reason_codes=None, properties=None):
    print(f"[on_publish] mid: {mid} reason: {reason_codes}")

def on_subscribe(client, userdata, mid, reason_codes, properties):
    print(f"[on_subscribe] mid: {mid} reason_codes: {reason_codes}")

def on_unsubscribe(client, userdata, mid, reason_codes, properties):
    print(f"[on_unsubscribe] mid: {mid} reason_codes: {reason_codes}")

def on_log(client, userdata, level, buf):
    print(f"[on_log] {buf}")

def on_socket_open(client, userdata, sock):
    print("[on_socket_open] Socket opened")

def on_socket_close(client, userdata, sock):
    print("[on_socket_close] Socket closed")

def on_socket_register_write(client, userdata, sock):
    print("[on_socket_register_write] Socket ready for write")

def on_socket_unregister_write(client, userdata, sock):
    print("[on_socket_unregister_write] Socket not ready for write")

# --- Client Setup ---

client = mqtt.Client(client_id=CLIENT_ID, callback_api_version=mqtt.CallbackAPIVersion.VERSION2)

client.on_connect = on_connect
client.on_connect_fail = on_connect_fail
client.on_disconnect = on_disconnect
client.on_message = on_message
client.on_publish = on_publish
client.on_subscribe = on_subscribe
client.on_unsubscribe = on_unsubscribe
client.on_log = on_log
client.on_socket_open = on_socket_open
client.on_socket_close = on_socket_close
client.on_socket_register_write = on_socket_register_write
client.on_socket_unregister_write = on_socket_unregister_write

# --- Start Client ---

client.connect(BROKER, PORT, 60)
client.loop_start()
```

    📶 [on_socket_open] Socket opened
    📝 [on_log] Sending CONNECT (u0, p0, wr0, wq0, wf0, c1, k60) client_id=b'jupyter-client-b11aed8c-5d98-4f04-a984-6a0c1d6a2978'
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write

    <MQTTErrorCode.MQTT_ERR_SUCCESS: 0>

    📝 [on_log] Received CONNACK (0, 0)
    ✅ [on_connect] Connected: Success
      ↪ session_present: False
    📝 [on_log] Sending SUBSCRIBE (d0, m1) [(b'#', 0)]
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received SUBACK
    🔔 [on_subscribe] mid: 1 reason_codes: [ReasonCode(Suback, 'Granted QoS 0')]
    📝 [on_log] Received PUBLISH (d0, q0, r1, m0), '/online', ...  (1 bytes)
    📩 [on_message] Topic: /online | Payload: 0
    📤 [on_publish] mid: 2 reason: Success
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PUBLISH (d0, q0, r0, m0), '/led', ...  (1 bytes)
    📩 [on_message] Topic: /led | Payload: 0
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📝 [on_log] Sending PINGREQ
    🖊️ [on_socket_register_write] Socket ready for write
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PINGRESP
    📤 [on_publish] mid: 3 reason: Success
    🛑 [on_socket_unregister_write] Socket not ready for write
    📝 [on_log] Received PUBLISH (d0, q0, r0, m0), '/led', ...  (1 bytes)
    📩 [on_message] Topic: /led | Payload: 0

``` python
for i in range(5):
    client.publish("test/topic", f"Message {i}")
```

``` python
client.publish("/led", f"0")
```

    📝 [on_log] Sending PUBLISH (d0, q0, r0, m3), 'b'/led'', ... (1 bytes)
    🖊️ [on_socket_register_write] Socket ready for write

    <paho.mqtt.client.MQTTMessageInfo>

``` python
import matplotlib.pyplot as plt
from IPython.display import clear_output

data = []

def on_message(client, userdata, msg):
    value = float(msg.payload.decode())
    data.append(value)
    
    clear_output(wait=True)
    plt.plot(data)
    plt.title("Live MQTT Plot")
    plt.xlabel("Message #")
    plt.ylabel("Value")
    plt.grid()
    plt.show()

client.on_message = on_message
result, mid = client.subscribe("sensor/temp")
if result == mqtt.MQTT_ERR_SUCCESS:
    print(f"Subscribed successfully! MID = {mid}")
else:
    print(f"Failed to subscribe. Error code: {result}")
```

    ✅ Subscribed successfully! MID = 14

Back to top
