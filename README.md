# Time-based Conflict Resolution Mechanism
For conflict mitigation, we used  time based conflcit mitigatio. mechanism. We implemented a time-based conflict resolution mechanism using a first-come, first-serve principle. RC messages are timestamped upon receipt and processed
Chronologically. In the CCA, new messages are compared with existing ones based on timestamps. This approach ensures organized handling of control decisions, with priority given to the earliest messages in the execution queue. The proposed
technique aims to prioritize control messages based on their arrival time. This approach may not be always suitable due to the differing operational timescales of various xApps. To tackle this one of the options is to implement a priority
system that considers the criticality and urgency of xApps and their control messages, ensuring more prompt handling of time-sensitive messages.


# ORAN SC RIC in Docker

This repository provides a minimal version of the **O-RAN Software Community (SC) Near-Real-time RIC** (`i-release`). It includes configuration files that enable the building and deploying of the SC RIC as a multi-container application using a single Docker command, without Kubernetes or Helm. Additionally, the repository features example monitoring and control xApps, which use the  ``E2SM_KPM`` and  ``E2SM_RC`` service modules, respectively.

## Repository Structure

This repository is organized as follows:

```
oran-sc-ric
├── e2-agents  -> Configs/code/etc required by E2 agents
│   └── srsRAN -> Config files to run srsRAN_Project gNB (with E2-Agent) with srsUE over zmq
│
├── ric
│   └── configs -> Config files for each SC RIC entity
│   └── images  -> Dockerfiles to build rtmgr_sim and ric-plt-xapp-frame-py images
│
├── xApps
│   └── python  -> Directory containing xApps written in Python
│
├── .env -> Default environment variables used to build and run SC RIC multi-docker application
├── create_ric_config_files.sh -> A script that creates config files for RIC entities.
├── docker-compose.yml -> A recipe used to build and run SC RIC multi-docker application
└── README.md
```

**Note 1:** We mostly use Docker images (i-release) provided in the [ORAN repository](https://nexus3.o-ran-sc.org/ ). However, we provide two Dockerfiles to build [`rtmgr_sim`](https://github.com/o-ran-sc/ric-plt-e2mgr/tree/i-release/tools/RoutingManagerSimulator) and [`ric-plt-xapp-frame-py`](https://github.com/o-ran-sc/ric-plt-xapp-frame-py/tree/i-release) images, as they are not available in the repository. Note that we build them with the code provided in the official [ORAN GitHub repository](https://github.com/o-ran-sc) and use branches related to `i-release`.

**Note 2:** There is no need to execute the `create_ric_config_files.sh` script, as default configs for each RIC entity are provided. However, this file might be used as a reference to see how the RIC entities are connected (i.e., one can check which IP addresses are provided for an individual entity).

## SC RIC Multi-Container Application

The SC RIC application contains the following services:

- `e2term`  &rarr; RIC E2 Termination Point that communicates with E2 agents over SCTP connections [[docs](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-e2/en/latest/), [code](https://github.com/o-ran-sc/ric-plt-e2)]
- `e2mgr` &rarr; E2 Manager responsible for maintaining other RIC entities and connections with E2 agents [[docs](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-e2mgr/en/latest/), [code](https://github.com/o-ran-sc/ric-plt-e2mgr)]
- `dbaas` &rarr; Redis Database acts as backend service for Shared Data Layer (SDL) [[docs](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-dbaas/en/latest/overview.html), [code](https://github.com/o-ran-sc/ric-plt-dbaas)]
- `rtmgr_sim` &rarr; Routing Manager Simulator used to mimic the real Routing Manager [[docs](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-rtmgr/en/latest/), [sim code](https://github.com/o-ran-sc/ric-plt-e2mgr/tree/i-release/tools/RoutingManagerSimulator), [code](https://github.com/o-ran-sc/ric-plt-rtmgr)]
- `submgr` &rarr; Subscription Manager responsible for managing E2 subscriptions from xApps to the E2 Node [[docs](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-submgr/en/latest/user-guide.html), [code](https://github.com/o-ran-sc/ric-plt-submgr)]
- `appmgr`  &rarr;  Application Manager responsible for deploying and managing RIC xApps [[docs](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-appmgr/en/latest/), [code](https://github.com/o-ran-sc/ric-plt-appmgr)]
- `python_xapp_runner`  &rarr; a container with xApp Python Framework to facilitate the development and running of xApps [[docs](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-xapp-frame-py/en/latest/), [code](https://github.com/o-ran-sc/ric-plt-xapp-frame-py)]

Each RIC entity is connected to [RIC Message Router](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-lib-rmr/en/latest/user-guide.html) (RMR), which acts as a peer-to-peer communication platform between entities. Therefore, each entity maintains a [routing table](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-lib-rmr/en/latest/rt_tables.html) where the routes are selected based on E2 message type.

Moreover, some entities communicate using REST API. For example, an xAPP sends a `RIC Subscription Request` message to the Subscription Manager using REST API, which translates it to an E2 message (following `E2AP` and `E2SM` ASN1 definitions) and sends it to E2 Termination (`e2term`) point. Upon a successful subscription, the `e2term` sends `RIC_INDICATION` messages directly to the xAPP over the RMR platform.  Details of this procedure can be found [here](https://docs.o-ran-sc.org/projects/o-ran-sc-ric-plt-submgr/en/latest/user-guide.html#architecture).

**Note 1:**  We use the Routing Manager Simulator (`rtmgr_sim`), as we pass a static routing table to all RIC entities (i.e., `ric/configs/routes.rtg`. The real Routing Manager (`rtmgr`) creates routes dynamically upon request from entities, however, it seems that the `rtmgr` requires Kubernetes and Helm to run. More investigation is needed to start it as a simple docker service without K8s and Helm.

**Note 2:** Currently, due to the usage of the static routing tables, we support multiple xApps in the following way. We add multiple endpoints in the routing entry for message type `12050` (i.e., `RIC_INDICATION`). Therefore, all `RIC_INDICATION` messages from E2 termination entity messages are sent to multiple endpoints (i.e., xApps), and then the indication messages are filtered by `Subscription ID` internally in each xApp.

## Example xApps

We provide two example [xApps](xApps/python/) designed to monitor any measurement metric exposed by an `E2SM_KPM` service module within an E2 Agent.  These xApps function by sending a `RIC Subscription Request` message, which includes a `RIC Subscription Details` Information Element crafted following the `E2SM_KPM` definition. Subsequently, they receive `RIC Indication Messages` containing measurement data adhering to the `E2SM_KPM_IndicationMessage` definition. In addition, we provide one example xApp that demonstrates the usage of the `E2SM_RC` service module.

The [simple_mon_xapp](xApps/python/simple_mon_xapp.py) serves as a basic example designed to illustrate the structure of an xApp.  It exclusively utilizes `E2SM-KPM-Report-Style-1`, enabling requests for any *E2-Node-Level* metric (i.e., aggregated metrics for the entire E2 node, for example, total DL throughput in a gNB node).

The [kpm_mon_xapp](xApps/python/kpm_mon_xapp.py) serves as a comprehensive E2SM-KPM monitor, facilitating all `E2SM-KPM-Report-Styles`(i.e., 1-5). Specifically, it allows requesting *E2-Node-Level* and *UE -Level* (e.g., individual UE DL throughput) metrics.

The [simple_rc_xapp](xApps/python/simple_rc_xapp.py) is a basic example designed to illustrate the usage of the RIC control API. So far, only Action ID 2 from RIC Control Style 2 (i.e., Slice-level PRB Quota control) is supported.

The [simple_xapp](xApps/python/simple_xapp.py) demonstrates how to use both E2SM-KPM monitoring and E2SM-RC control modules together.

Additionally, we provide a handy [library](xApps/python/lib/) designed to streamline xApp development. This library focuses on separating reusable code, such as communication protocols and message encoding/decoding, resulting in significantly simplified xApp implementations.

Specifically, we provide:

- [xAppBase](xApps/python/lib/xAppBase.py) class, which abstracts the intricacies of communication via the RMR platform and REST API.
- [e2sm_kpm_module](xApps/python/lib/e2sm_kpm_module.py) class, which offers a user-friendly subscription API and manages the encoding/decoding of E2SM_KPM ASN1 messages.
- [e2sm_rc_module](xApps/python/lib/e2sm_rc_module.py) class, which offers a user-friendly subscription API and manages the encoding/decoding of E2SM_RC ASN1 messages.

All files are written in Python and serve as excellent starting points for creating new xApps.

## Quick start
#### 1. SC RIC platform

To launch the SC RIC, please run the following command from the `oran-sc-ric` directory:

```bash
docker compose up
```

- To force a new build of the containers, please add a `--build` flag at the end of the previous command.
- To run it in background, please add a `-d` flag at the end of the previous command.
- For more options, check `docker compose up --help`

**Note:** Running this command for the first time may take up to a few minutes, as multiple Docker images have to be downloaded and/or built. A subsequent command execution (i.e., after the environment is ready) starts the RIC in seconds.

#### 2.  5G RAN
We set up an end-to-end 5G network using the **srsRAN_Project gNB** [[docs](https://docs.srsran.com/projects/project/en/latest/),[code](https://github.com/srsran/srsRAN_Project/)] (that is equipped with an E2 agent) and **srsUE** from **srsRAN-4g** project [[docs](https://docs.srsran.com/projects/4g/en/latest/),[code](https://github.com/srsran/srsRAN_4G)]. Please follow the official installation guidelines and remember to compile both projects with **ZeroMQ** support.

We follow this [application note](https://docs.srsran.com/projects/project/en/latest/tutorials/source/flexric/source/index.html), but use the SC RIC instead of [Flexric](https://gitlab.eurecom.fr/mosaic5g/flexric). To this end, we execute gNB and srsUE with the configs provided in the `./e2-agents/srsRAN` directory (gNB config differs only with the IP address of the RIC compared to the config from the tutorial). Note that, we use ZMQ-based RF devices for emulation of the wireless transmission between gNB and UE, therefore the entire RAN setup can be run on a single host machine.

2.1. Start Core Network (here [Open5GS](https://open5gs.org/open5gs/docs/))
```bash
cd  ./srsRAN_Project/docker/
docker compose up --build 5gc
```
2.2. Start gNB:
```bash
cd  ./srsRAN_Project/build/apps/gnb/
sudo ./gnb -c ~/oran-sc-ric/e2-agents/srsRAN/gnb_zmq.yaml
```
The gNB should connect to both the core network and the RIC.  
**Note:** The RIC uses 60s time-to-wait. Therefore, after disconnecting from RIC, an E2 agent (inside gNB) has to wait 60s before trying to connect again. Otherwise, the RIC sends an `E2 SETUP FAILURE` message and gNB is not connected to the RIC.

#### 2.3. COTS UE   https://docs.srsran.com/projects/project/en/latest/tutorials/source/cotsUE/source/index.html
Start   COTS UE:
To connect the COTS UE to the network the following steps must be taken once the phone and network have been correctly configured:

Run the gNB and ensure it is correctly connected to the core

Search for the network from the UE

Select and connect to the network

Verify the attach

[once the started the core will detect the plmn address and register it and an ip address will be assigned]

In the ue terminal start the iperf server by running the following command:
#### iperf3 -s

In another terminal start iperf to generate downlink throughput by running this command with the ipv4 address generated in the core
```bash
sudo iperf3 -c 10.45.0.2 -i 1 -t 5000 -u -b 10M
```
#### Grafana Metrics GUI

To visualize the downlink throughput we can enable the grafana dashboard from srsran project directory  [https://docs.srsran.com/projects/project/en/latest/user_manuals/source/grafana_gui.html]


#### 4. Example xApp

The xApp in this demo is designed to send rc message to the e2 node. Two custom xapp are being used in this experiment. Xapp-1 sends rc message to allocate higher resources (prb) to the base station and xapp-2 sends rc message to allocate minimum resources to the base station. When both xapp is running concurrently it results in an unstable network at the base station resulting in a direct conflict

To start the xapp1, which will alocate prb to  maximum ratio, run the following command
```bash
 sudo docker compose exec python_xapp_runner ./simple_xapp_12.py --http_server_port 8091 --rmr_port 4560 --e2_node_id gnbd_001_001_00019b_0 --ran_func_id 3 --ue_id 0 --xapp_id xApp2```
```
#### 5. Start Another xapp
To Start xapp2, which will allocate prb to minimum ratio, run the following command .
If we start the xapp 2 , its decision will override on the xapp-1 s decision which will reflect a fluctuation in downlink bitrate.


```bash
 sudo docker compose exec python_xapp_runner ./simple_xapp_13.py --http_server_port 8090 --rmr_port 4560 --e2_node_id gnbd_001_001_00019b_0 --ran_func_id 3 --ue_id 0 --xapp_id xApp1```

```
Enabling gNB console trace (with `t`) allows the monitoring of changes in the downlink (DL) user equipment (UE) data rate.

#### 6. Conflict Detection
In another Terminal run the following command to see if the central controller detcting the conflcit uplon starting the 2nd xapp
        
```bash
 sudo python3 ./central_controller_cd.py
```
The terminal will print this after detecting conflict
```bash
Checking for conflicts upon onboarding xApp xApp1
Logging message from xApp xApp1 at 2025-05-09 11:38:25.396068
Checking for conflicts among 1 recent messages
  
Checking for conflicts upon onboarding xApp xApp2
Logging message from xApp xApp2 at 2025-05-09 11:38:25.434621
Checking for conflicts among 2 recent messages
Conflict detected between messages from  xApp1 and  xApp2
  
Conflict detected. Both  xApp1 and  xApp2 sent conflicting messages.
Initializing Conflict Mitigation Module
Buffering message from  xApp2 for later execution.
Executing buffered message from xApp xApp2 after delay.
Initializing Conflict Mitigation Module

```
#### 7. Conflict Mitigation
After detecting the conflict we need to initialize the mitigation by clicking the following command

```bash
cd oran-sc-ric/
 sudo docker compose exec python_xapp_runner ./resolution.py
```
This mitigation technique uses a first come first serve basis. The controller will log the timestamp of the arrival of each xapp message. It will execute the first xapps message and hold the 2nd xapp for a certain period of time. In this demo, we used 10 second time window which can be tunable according to the user demand. After 10 seconds the controller will execute the 2nd xapps message and hold the first one. With this method, the controller will execute both xapps decision without degrading the network performance
## xApp Development

The `xApps/python` directory is mounted into the `python_xapp_runner` container, therefore one can develop the xApp on a local machine, put it in this directory, and execute it inside the docker without restarting the SC RIC platform.

The `ric-plt-xapp-frame-py` Python module is installed in development mode (i.e., *editable installation* with `pip install -e ...`).  If you want to develop/modify the module and use your modified local version inside the `python_xapp_runner` container, please uncomment the following lines inside the `docker-compose.yml` and provide a path to your local copy of the `ric-plt-xapp-frame-py` code:

```yaml
  python_xapp_runner:
    ...
    volumes:
      ...
      # Uncomment if you want to use your local ric-plt-xapp-frame-py copy inside the container
      #- type: bind
      #  source: ./Path/to/your/local/ric-plt-xapp-frame-py
      #  target: /opt/ric-plt-xapp-frame-py
```

**Note:** It might be necessary to apply our patch (available in `ric/images/ric-plt-xapp-frame-py/`) to your local `ric-plt-xapp-frame-py` code.

## Troubleshooting

If you're not familiarized with `docker compose` tool, it is recommended to check its [website](https://docs.docker.com/compose/) and `docker compose --help` output.

#### Run a single service

Instead of running all services provided, a partial run is allowed by doing:

```bash
docker compose up <service_name>
```
Note that any service declared in the ``docker-compose.yml`` file can be started standalone.

#### Checking services' output
If a service is started in the background, one can check its output with the following command:

```bash
docker compose logs [OPTIONS] [SERVICE...]
```

- For more options, check `docker compose logs --help`

#### To stop RIC:

When running in the foreground, the entire SC RIC multi-container application can be stopped with the `Ctrl+C` signal.

When running in the background, please execute:

```bash
docker compose down
```
For more options, check `docker compose down --help`

#### Monitor RIC communication:

You can use Wireshark to track all messages exchanged among RIC entities. To this end, you need to start sniffing on the network bridge interface created by Docker for the RIC. Please execute `ifconfig` to check which interface has the IP address used by the RIC network (i.e., by default it should be `10.0.2.1`).

If you want to monitor only E2AP packets then set the Wireshark filter to `e2ap`, then right-click on any packet, then go to `Decode As..` and set `Current` to `E2AP`.

**Note:**  You need at least Wireshark version 4.0.7 to display content of E2AP packets correctly. It should be the default version available for installation in Ubuntu 23.04. However, it is recommended to use higher versions (e.g., Wireshark version 4.1.0 (v4.1.0rc0-3390-g4f4a54e6d3f9)) that have to be built from the [source code](https://github.com/wireshark/wireshark).

