# Influx

Time series Backend

Author

Benedict Thekkel

## Installation

Inside WSL

``` sh
# Ubuntu/Debian AMD64
curl -O https://dl.influxdata.com/influxdb/releases/influxdb2_2.7.3-1_amd64.deb
sudo dpkg -i influxdb2_2.7.3-1_amd64.deb
```

## To view all Services

``` sh
systemctl --type=service --state=running
```

``` sh
pstree | head -5
```

``` python
!ps
```

        PID TTY          TIME CMD
    2048646 pts/18   00:00:00 ps

``` python
!pstree
```

    systemd─┬─2*[agetty]
            ├─cron
            ├─dbus-daemon
            ├─grafana───20*[{grafana}]
            ├─influxd───53*[{influxd}]
            ├─init-systemd(Ub─┬─SessionLeader───Relay(400)─┬─bash───tmux: client
            │                 │                            ├─python───22*[{python}]
            │                 │                            └─tmux: server─┬─bash───+
            │                 │                                           ├─6*[bash+
            │                 │                                           ├─bash───+
            │                 │                                           └─bash───+
            │                 ├─SessionLeader───Relay(707)───bash
            │                 ├─SessionLeader───Relay(8620)───bash───tmux: client
            │                 ├─SessionLeader───Relay(1997513)───bash───tmux: clien+
            │                 ├─init───{init}
            │                 ├─login───bash
            │                 └─{init-systemd(Ub}
            ├─networkd-dispat
            ├─packagekitd───2*[{packagekitd}]
            ├─polkitd───2*[{polkitd}]
            ├─rsyslogd───3*[{rsyslogd}]
            ├─smtpd───6*[smtpd]
            ├─snapd───18*[{snapd}]
            ├─16*[snapfuse]
            ├─sshd
            ├─subiquity-serve───python3.10─┬─python3
            │                              └─5*[{python3.10}]
            ├─systemd───(sd-pam)
            ├─systemd-journal
            ├─systemd-logind
            ├─systemd-resolve
            ├─systemd-udevd───2*[systemd-udevd]
            └─unattended-upgr───{unattended-upgr}

## Server Settings

### To start influx server

``` sh
sudo service influxdb start --http-bind-address :8080
```

### Show Status of influx server

``` sh
sudo service influxdb status
```

### To Stop influx server

``` sh
sudo service influxdb stop
```

## InfluxDB OSS URLs

``` sh
http://localhost:8086/
```

## T0 change ports

``` sh
influxd --http-bind-address :8080
```

## To setup influx server

> First step, Must do

------------------------------------------------------------------------

``` sh
influx setup \
  --username bthek1 \
  --password <password> \
  --token TOKEN \
  --org ORGANIZATION_NAME \
  --bucket BUCKET_NAME \
  --force
```

## Organisation Settings

### To show organisation list

``` sh
influx org ls
```

## Bucket Settings

### To show Bucket list

``` sh
influx bucket ls
```

### To create bucket

``` sh
influx bucket create --name <name> -c <config-file>
```

### To enter data to bucket

``` sh
influx write --bucket sample-bucket --url https://influx-testdata.s3.amazonaws.com/air-sensor-data-annotated.csv
```

``` sh
influx write -b energy_data -f 'energy.lp'
```

### To query data from influx

``` sh
influx query 'from(bucket:\"sample-bucket\") |> range(start:-30m)'
```

``` sh
influx query 'from(bucket: "sample-bucket")
  |> range(start:-4h , stop: -0m)
  |> filter(fn: (r) => r["_measurement"] == "airSensors")
  |> filter(fn: (r) => r["_field"] == "temperature")
  |> aggregateWindow(every: 1m, fn: mean, createEmpty: false)
  |> yield(name: "mean")'
```

## User settings

### To create new user

``` sh
influx user create -n <username> -p <password> -o <org-name> 
```

``` sh
influx user create -n ben -p Asdf,mnb1234 -o phisaver
```

### To show user list

``` sh
influx user ls
```

### To delete user

``` sh
influx user delete -i <id>
```

### To delete user

``` sh
influx user update -i <user-id> -n <new-username>
```

## Task

``` python
import influxdata/influxdb/sample


option task = {
  name: "Collect air sensor sample data",
  every: '15m',
}

sample.data(set: "airSensor")
    |> to(bucket: "example-bucket")
```

## Python Version

### Create config.ini file

``` bash
[APP]
INFLUX_URL = <INFLUX_URL>
INFLUX_TOKEN = <INFLUX_TOKEN>
INFLUX_ORG = <INFLUX_ORG_ID>
INFLUX_BUCKET = iot_center
INFLUX_BUCKET_AUTH = iot_center_devices
```

``` python
# Import the dependencies.
import configparser
from datetime import datetime
from uuid import uuid4

# Import client library classes.
from influxdb_client import Authorization, InfluxDBClient, Permission, PermissionResource, Point, WriteOptions

from influxdb_client.client.write_api import SYNCHRONOUS
import influxdb_client

# Get the configuration key-value pairs.

config = configparser.ConfigParser()
config.read('config.ini')
```

    ['config.ini']

### Connect to server using InfluxDBClient

``` python
client = InfluxDBClient(url=config.get('APP', 'INFLUX_URL'),
                        token=config.get('APP', 'INFLUX_TOKEN'),
                        org=config.get('APP', 'INFLUX_ORG'),)
```

``` python
client.ping()
```

    True

### Auth: create, list, delete

``` python
authorization_api = client.authorizations_api()
auth_list = authorization_api.find_authorizations()

for auth in auth_list:
    print(auth.token)
```

    p_5lZramPcqYC4EPNRVpRhAFs4RvWc8cPXtt9exsIPH8ybJP3lKlf5KMKjLf2ueHFlcmo45ZBxeeU3ZtKQ-LDQ==
    CeAAU2CprLUJoammpG9G7JuGTZ2qosK_shCBdXZvnIai46GFf7VB8jfd8XV6yeAeRjAc8nwAYkyhYL5BhDji2g==
    U1HdRByFCwZJhoNaKqVg9tja0zHw23TO7-pUxSSFhzDxt740WZ1DRUZoRWx6Brs5lDRBJqU-bB11kkVGBxtNfg==
    IPPdYDQH2s43p7jCro1SI_oJ8aGhFjkDEpgMSFuGs-vXuKnOMy1mvQzVLBoth2tKmvJGpy6ePo5c93sO8xTUkg==

### Bucket: create, list, delete

``` python
buckets_api = client.buckets_api()
bucket_list = buckets_api.find_buckets()
```

``` python
for bucket in bucket_list.buckets:
    print(bucket.name)
```

    _tasks
    sample-bucket
    _monitoring
    energy_data
    power_data
    test
    Weather

``` python
client = InfluxDBClient(url=config.get('APP', 'INFLUX_URL'),
                        token=config.get('APP', 'INFLUX_TOKEN'),
                        org=config.get('APP', 'INFLUX_ORG'),)

buckets_api = client.buckets_api()
bucket_list = buckets_api.find_buckets()
bucket_name = 'solar_forecast'

if bucket_name not in [bucket.name for bucket in bucket_list.buckets]:
    buckets_api.create_bucket(bucket_name = bucket_name)
else:
    print('bucket already exists')
```

    bucket already exists

``` python
# Specify the name of the bucket you want to delete
bucket_name = "your_bucket_name"

try:
    # Delete the bucket
    client.buckets_api().delete_bucket(bucket_name=bucket_name)
    print(f"Bucket '{bucket_name}' deleted successfully.")
except ApiException as e:
    print(f"Failed to delete bucket '{bucket_name}': {e}")
```

### Organization: create, list, delete

``` python
ori_api = client.organizations_api()
organization_list = ori_api.find_organizations()
```

``` python
for ori in organization_list:
    print(ori.name)
```

    Org1

### User: create, list, delete

``` python
users_api = client.users_api()
```

``` python
users_list = users_api.find_users()
users_list
```

    {'links': {'_self': '/api/v2/users'},
     'users': [{'id': '0ca78064bac9d000',
                'links': {'_self': '/api/v2/users/0ca78064bac9d000'},
                'name': 'ben',
                'status': 'active'}]}

### Query: create, list, delete

``` python
import pandas as pd
```

``` python
query_api = client.query_api()

query = '''from(bucket: "sample-bucket")
  |> range(start: 2024-02-28T03:50:00.000Z, stop: 2024-02-28T04:50:00.000Z)
  |> filter(fn: (r) => r["_measurement"] == "airSensors")
  |> filter(fn: (r) => r["_field"] == "humidity")
  |> filter(fn: (r) => r["sensor_id"] == "TLM0101" or r["sensor_id"] == "TLM0102")
  |> aggregateWindow(every: 1s, fn: mean, createEmpty: false)
  |> yield(name: "mean") 
  |> pivot(rowKey:["_time"], columnKey: ["_field"], valueColumn: "_value")'''
```

``` python
data = query_api.query_data_frame(org=config.get('APP', 'INFLUX_ORG'), query=query)
```

``` python
data1 = data[0]
```

``` python
data1
```

|  | result | table | \_start | \_stop | \_time | \_value | \_field | \_measurement | sensor_id |
|----|----|----|----|----|----|----|----|----|----|
| 0 | mean | 0 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 03:50:05+00:00 | 34.889344 | humidity | airSensors | TLM0101 |
| 1 | mean | 0 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 03:50:15+00:00 | 34.919323 | humidity | airSensors | TLM0101 |
| 2 | mean | 0 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 03:50:25+00:00 | 34.929411 | humidity | airSensors | TLM0101 |
| 3 | mean | 0 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 03:50:35+00:00 | 34.920287 | humidity | airSensors | TLM0101 |
| 4 | mean | 0 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 03:50:45+00:00 | 34.969323 | humidity | airSensors | TLM0101 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |
| 705 | mean | 1 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 04:48:25+00:00 | 35.477672 | humidity | airSensors | TLM0102 |
| 706 | mean | 1 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 04:48:35+00:00 | 35.434914 | humidity | airSensors | TLM0102 |
| 707 | mean | 1 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 04:48:45+00:00 | 35.466550 | humidity | airSensors | TLM0102 |
| 708 | mean | 1 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 04:48:55+00:00 | 35.433109 | humidity | airSensors | TLM0102 |
| 709 | mean | 1 | 2024-02-28 03:50:00+00:00 | 2024-02-28 04:50:00+00:00 | 2024-02-28 04:49:05+00:00 | 35.482699 | humidity | airSensors | TLM0102 |

710 rows × 9 columns

``` python
data1.loc[1]
```

    result                               mean
    table                                   0
    _start          2024-02-28 03:50:00+00:00
    _stop           2024-02-28 04:50:00+00:00
    _time           2024-02-28 03:50:15+00:00
    _value                          34.919323
    _field                           humidity
    _measurement                   airSensors
    sensor_id                         TLM0101
    Name: 1, dtype: object

### Write: create, list, delete

``` python
from datetime import UTC, datetime
```

``` python
write_Api = client.write_api(write_options=SYNCHRONOUS)
```

``` python
dt = '2024-03-01 05:20:09'
date = pd.to_datetime(dt).strftime('%Y-%m-%dT%H:%M:%SZ')
date
```

    '2024-03-01T05:20:09Z'

``` python
timestamp = datetime.now(UTC).strftime('%Y-%m-%dT%H:%M:%SZ')
timestamp
```

    '2024-02-29T09:42:00Z'

``` python
dictionary = {"measurement": "h2o_feet",
              "tags": {"location": "us-west"},
              "fields": {"level": 150},
              "time": date
             }
dictionary
```

    {'measurement': 'h2o_feet',
     'tags': {'location': 'us-west'},
     'fields': {'level': 150},
     'time': '2024-03-01T05:20:09Z'}

``` python
write_Api.write(bucket="test",org="Org1", record=dictionary)
```

``` python
path = '/home/ben/BENEDICT_Only/Benedict_Projects/Intership/Phisaver/data_phisaver/energy.lp'

# Open the .lp file
with open(path, 'r') as file:
    # Read the first 5 lines
    for line in file:
        write_Api.write(bucket=bucket_name, 
                        org="Org1",
                        record=line.strip())
```

``` python
with open(path, 'r') as file:
    lp_content = file.read()
```

``` python
for line in tqdm(text.split('\n')):
        write_Api.write(bucket='test', 
                        org="Org1",
                        record=line.strip())
```

Back to top
