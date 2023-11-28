<!--
SPDX-FileCopyrightText: 2023 Contributors to the Eclipse Foundation

See the NOTICE file(s) distributed with this work for additional
information regarding copyright ownership.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

SPDX-License-Identifier: Apache-2.0
-->

# Setup with Eclipse Leda

The components of the Fleet Management blueprint fall into two categories: the
*KUKSA.val Databroker*, *CSV Provider* and *FMS Forwarder* components are
supposed to run in the vehicle, whereas the remaining components are supposed
to run in a (cloud) back end to which the vehicle is connected via (public
internet) networking infrastructure.

All of the components can be run on a single Docker host as described in the
[top level README](../README.md). However, the following sections describe a
more realistic deployment scenario in which [Eclipse Leda](https://eclipse-leda.github.io/leda/)
is used as the vehicle runtime environment.

This guide was tested with the
[0.1.0-M3](https://github.com/eclipse-leda/leda-distro/releases/tag/v0.1.0-M3)
release of Eclipse Leda.

# Start Back End Components

The containers for the back end components are run on the local host using Docker Compose:

```sh
# in this repository's root folder
docker compose -f fms-blueprint-compose.yaml up influxdb grafana fms-server --detach
```

# Start In-Vehicle Coponents

In the setup described here, the containers for the in-vehicle components are
deployed to a Leda instance running on the same (local) host as the back-end
components. The *FMS Forwarder* running in Leda then connects to the *influxdb*
server managed by Docker Compose.

Please refer to [Leda's Getting Started](https://eclipse-leda.github.io/leda/docs/general-usage/)
guide for setting up a Leda instance.

## Stop default Containers

Leda comes with a set of default containers (including Kuksa.val Databroker) that
are managed using Eclipse Kanto's *container-manager*. These containers are defined
by means of JSON manifest files in Leda's `/data/var/containers/mainfests` folder.

We will stop, remove and disable most of Leda's default containers and deploy the containers
of the Fleet Management Blueprint via Kanto's Update Manager as described below.

```sh
# in Leda instance's /data/var/containers/mainfests folder
tar cf manifests.orig.tar * 
kanto-cm remove --force -n feedercan
mv feedercan.json feedercan.json.disabled
kanto-cm remove --force -n feedergps
mv feedergps.json feedergps.json.disabled
kanto-cm remove --force -n hvacservice-example
mv hvac.json hvac.json.disabled
kanto-cm remove --force -n node-red-example
mv node-red.json node-red.json.disabled
kanto-cm remove --force -n seatservice-example
mv seatservice.json seatservice.json.disabled
kanto-cm remove --force -n databroker
mv databroker.json databroker.json.disabled
```

## Deploy in-vehicle Blueprint Components

Some of the Fleet Management blueprint containers require access to configuration files
that are not present in Leda, e.g. the FMS-specific VSS definitions.

Create the folders in Leda from which the containers can mount these files:

```sh
# in Leda instance
mkdir -p /data/usr/fms/csv /data/usr/fms/databroker /data/usr/fms/forwarder
```

Now copy the files required by the containers to the Leda instance:

```sh
# on the (local) host that you have started the back end components on
docker exec -it influxDB cat /tmp/out/fms-demo.token > /tmp/influxdb.token
```

```sh
# in this repository's root folder
# The CSV Provider needs acces to the recording of the signals to play back.
scp -P 2222 csv-provider/signalsFmsRecording.csv root@127.0.0.1:/data/usr/fms/csv
# The KUKSA.val Databroker needs to be configured with the VSS definition that also contains
# the fleet management specific signals. The default definition file that comes with Leda does
# not contain these.
scp -P 2222 spec/overlay/vss.json root@127.0.0.1:/data/usr/fms/databroker
# The FMS Forwarder needs to read the token required for authenticating to influxdb.
scp -P 2222 /tmp/influxdb.token root@127.0.0.1:/data/usr/fms/forwarder
```

Finally, apply the desired state to the Update Manager running on Leda in order to
trigger execution of the in-vehicle components:

```sh
# in this repository's root folder
mosquitto_pub -h localhost -f leda/fms-desired-state.json -t vehicleupdate/desiredstate
```

> :bulb: **Tip: Track Application of Desired State**
> The Kanto Update Manager reports the progress of applying the desired state by means of
> messages being published to an MQTT topic. In order to track the progress, run the following
> command in a separate terminal *before* publishing the desired state:
> ```sh
> mosquitto_sub -h localhost -t vehicleupdate/desiredstatefeedback
> ```
> There are other topics you can use to e.g. get notified about the current state. Please refer
> to the [Kanto Update Manager documentation](https://github.com/eclipse-kanto/update-manager/tree/main/docs)
> for details.

# Run in-vehicle Components on a separate Node

If you want to run the in-vehicle and the back-end components on different devices,
e.g. a RaspberryPi 4 and a developer machine, you need to replace the network address
`127.0.0.1` in the `scp` commands with the respective IP address of the in-vehicle device.
You also need to adapt the IP address of the influx database in the container manifest
for the *FMS-Forwarder* (`config.env.INFLUXDB_URI` in `data/var/containers/manifests/fms-forwarder.json`).
