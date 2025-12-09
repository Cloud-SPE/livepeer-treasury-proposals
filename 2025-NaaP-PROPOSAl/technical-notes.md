# Overview
[Livepeer Naap Overview](https://www.notion.so/livepeer/Livepeer-Network-as-a-Product-2a30a348568780919946f80211e326ab)

[Livepeer NaaP MVP](https://www.notion.so/livepeer/Network-as-a-Product-MVP-SLA-Metrics-Analytics-Infra-2a70a3485687802ebbdad8a1a501827a)

# Architecture

<img src="Analytics_Architecture.png">
* Event Schema?
* Data Validation?

# Metrics To Collect

[Livepeer NaaP Metrics Catalog](https://www.notion.so/livepeer/6914345d128b477d81ee48409a6a9c90?v=fcb9e18cfe2c4746860c58b16a48c8b2)

## High
* CUDA Utilization Efficiency (CUE)
* E2E Stream Latency
* Prompt-To-First-Frame Latency

## Medium 
* Bandwidth(upload/download)
* Jitter Coefficent
* Failure Rate
* Swap Rate

## Low
* Startup Time
* Output FPS


## Future Metrics Collection
Ideas: metrics that fulfills their narrative - "realtime, scalable, leading edge, decentralized AI video is available on livepeer, which is growing in fees by the day"

* Price related????
    * how much is one getting paid per hour given a certain model, gpu, and mean 'latency score' binned by certain views like stake.
* how many jobs were running in the network
* Operating System Metrics 
    - CPU Type / Usage
    - RAM Type/Amount/Usage
    - Disk Amount/Usage
    - GPU Type/VRAM/Usage
* what else?????

# Streamr.Network Integration
[Streamr docs](https://docs.streamr.network/)

## Producer

## Consumer

## Historical Storage vs Realtime 
[Streamr Storage](https://docs.streamr.network/usage/streams/store-and-retrieve) 

[Storage Registry Smart Contract Polygon](https://polygonscan.com/address/0x080F34fec2bc33928999Ea9e39ADc798bEF3E0d6#readProxyContract)


* Cloud will NOT use storage nodes (unless we can define a true Streamr "storage node"). Otherwise, CloudSPE will collect all stream data and store to parquet (or some other format) for historical archival

## Access Control
[Streamr Permissions](https://docs.streamr.network/usage/streams/permissions)

* Should Streamr Stream be owned by Livepeer Foundation????

https://docs.streamr.network/usage/streams/signature-verification/#smart-contract-pubsub-erc-1271

## Partitions
[Streamr Partitions](https://docs.streamr.network/usage/streams/partitioning)

* Streamr advises to partition when messages throughput > 100 messages/second
    * what partition keys?


# go-livepeer Integration

[Lisbon Portual Hackathon Changes](https://github.com/livepeer/go-livepeer/pull/3774)

## Metrics Collection

* Census.go??

* Open Pool "Event Tracker" - incision points

* AI Runner / Transcoder metrics need to funnel into the Orchestrator for publishing. Otherwise access control could be problematic.

* How will access control be granted? we cannot require the Orch ETH Key to be present. How to "delegate" the Orch's writes to another wallet?

## Local Streamr PUblisher
* How to ensure access control?
* 

## Prior Metrics Work
* [AI Stream Test](https://github.com/livepeer/go-livepeer/pull/3241) changes need to be merged
    * need to fix the "Network Capabilities" endpoint logic to use brad's logic

# Other Resources

[Evan's Unified Network Pipeline Notes](https://livepeer.notion.site/Livepeer-Hackathon-Unified-Network-Data-Pipeline-2860a348568780b987bed752e2220c63#2f1532348d304be2943e3a93e4650af9)


## SAMPLE PAYLOADS

### existing go-livepeer messages - including Open Pool

```json
[
  {
    "id": "b6efce36-6ee9-4a5d-b2f2-715efcd4b32d",
    "type": "node_trace",
    "timestamp": "2025-12-05T16:24:24.271818368Z",
    "payload": "gateway-reset"
  }
]
```

```json
[
  {
    "id": "b68ff76d-abd9-4046-b17a-5b300c8d6407",
    "type": "node_trace",
    "timestamp": "2025-12-05T16:24:26.646101802Z",
    "payload": "orchestrator-reset"
  }
]
```

```json
[
  {
    "id": "6993003f-f2ca-434c-bee7-950562a27e15",
    "type": "worker-connected",
    "timestamp": "2025-12-05T16:24:31.228265243Z",
    "payload": {
      "connection": "127.0.0.1:46800",
      "ethAddress": "0x5263E0Ce3a97B634D8828CE4337aD0F70B30B077"
    }
  }
]
```

```json
[
  {
    "id": "0224e0ba-0c91-4eda-933f-96b31bcf9e34",
    "type": "node_trace",
    "timestamp": "2025-12-05T16:24:31.226345484Z",
    "payload": "transcoder-reset"
  }
]
```

```json
[
  {
    "id": "fa3eb4ca-d9a7-41de-a552-69c427582381",
    "type": "discovery_results",
    "timestamp": "2025-12-05T16:24:36.844802833Z",
    "payload": [
      {
        "address": "0x52cf2968b3dc6016778742d3539449a6597d5954",
        "latency_ms": "93",
        "url": "https://localhost:18935"
      }
    ]
  }
]
```

```json

[
  {
    "id": "f6643ebc-3d34-4ed1-8eb9-595228d5264b",
    "type": "network_capabilities",
    "timestamp": "2025-12-05T16:24:46.259972687Z",
    "payload": [
      {
        "address": "0x5263E0Ce3a97B634D8828CE4337aD0F70B30B077",
        "local_address": "0x52CF2968b3DC6016778742D3539449a6597D5954",
        "orch_uri": "https://localhost:18935",
        "capabilities": {
          "bitstring": [
            94422687743
          ],
          "capacities": {
            "0": 2,
            "1": 2,
            "2": 2,
            "3": 2,
            "4": 2,
            "5": 2,
            "6": 2,
            "7": 2,
            "8": 2,
            "9": 2,
            "10": 2,
            "11": 2,
            "12": 2,
            "14": 2,
            "15": 1,
            "16": 1,
            "17": 1,
            "18": 1,
            "26": 2,
            "27": 1,
            "28": 1,
            "29": 1,
            "30": 1,
            "31": 1,
            "32": 1,
            "34": 1,
            "36": 1
          },
          "version": "0.8.3",
          "constraints": {
            "minVersion": "0.8.3"
          }
        },
        "capabilities_prices": null,
        "hardware": null
      }
    ]
  }
]
```

```json
[
  {
    "id": "e041c365-7339-47f5-95fc-a1616359b263",
    "type": "create_new_payment",
    "timestamp": "2025-12-05T16:24:49.259990375Z",
    "payload": {
      "capability": "",
      "clientIP": "",
      "faceValue": "3400000000000000 WEI",
      "manifestID": "mike-test",
      "numTickets": "1",
      "orchestrator": "https://localhost:18935",
      "price": "137.038 wei/pixel",
      "recipient": "0x5263E0Ce3a97B634D8828CE4337aD0F70B30B077",
      "requestID": "",
      "sender": "0x5aE4E42dB3671370a0c25AfF451E7482aAEc3D0B",
      "sessionID": "3198bee9",
      "winProb": "0.0000147059"
    }
  }
]

```

```json
[
  {
    "id": "9255b365-50b3-4620-b6ea-ff5a7f727506",
    "type": "job-processed",
    "timestamp": "2025-12-05T16:24:50.197195053Z",
    "payload": {
      "computeUnits": 41580000,
      "duration": 4167,
      "ethAddress": "0x5263E0Ce3a97B634D8828CE4337aD0F70B30B077",
      "fees": 5696460000,
      "pricePerComputeUnit": 137,
      "realTimeRatio": 4,
      "responseTime": 845
    }
  }
]

```

```json
[
  {
    "id": "8c513c53-98a5-4b48-9b78-d34b2fc41a6e",
    "type": "discovery_results",
    "timestamp": "2025-12-05T16:24:50.38026783Z",
    "payload": [
      {
        "address": "0x52cf2968b3dc6016778742d3539449a6597d5954",
        "latency_ms": "1",
        "url": "https://localhost:18935"
      }
    ]
  }
]

```

```json
[
  {
    "id": "d5d309d6-bf2c-45db-8e63-4781e4778f6a",
    "type": "worker-disconnected",
    "timestamp": "2025-12-05T16:28:26.112684001Z",
    "payload": {
      "connection": "127.0.0.1:46800",
      "ethAddress": "0x5263E0Ce3a97B634D8828CE4337aD0F70B30B077"
    }
  }
]
```

# OPEN ITEMS/Questions

## notes about go-livepeer integration

* (DONE) pool event tracker and hackathon event publisher need to be merged
* streamr support multiple protocols (mqtt,http,web socket, etc...) - do we enable all these in go-livepeer??
* when the stream node is offline, the Orch/GTateway/Trans/Worker should work without it... WHAT HAPPENS to event data????
* when the gateway and orchestrator shutdown (abruptly ) - NO shutdown message is logged to the STREAMR pipe
* need to include MORE data with each message to streamr 
* orchestrator and gateways messages are generic and cannot be differentiated from other orch/gateways/etc...
* streamer publisher needs to have accesscontrl that allows for ANY orch or gateway to publish to streamr streams.
    * the current impl requires a centralized publsher with streamr private key. 
    * need some sort of "registration" mechanisum that allows the orch private key to "delegate" an ETH wallet (private key) to allow publishing on their behalf.
    * streamr publisher needs ACL to enable orchs/ gateways that are "active" based on livepeer smart contracts

## METRICS COLLECTION QUESTIONS

See Comments in the 
https://www.notion.so/livepeer/Network-as-a-Product-MVP-SLA-Metrics-Analytics-Infra-2a70a3485687802ebbdad8a1a501827a


General Themes

* Josh needs to understand how Latency is measured. 
* Josh says swap rate is already measured, but needs to work on the swap reasons. here is an example: https://eu-metrics-monitoring.livepeer.live/grafana/d/c711a631-97bf-4c75-b640-aae699d8a280/ai-swaps?orgId=1&from=now-24h&to=now&timezone=utc

* Josh wants to understand the architectural decision of not using Kafka.
  
  * Right now our Kafka metrics come from the gateway. Are we planning to keep this architecture, or push additional metrics through the orchestrator itself?
  * is the raw data ingested by streamr meant to be read by anyone, or is it only for the SPE running the metrics collection so they can transform it into something more useful and public facing? If the latter (SPE only) then I wonder what streamr actually offers; the SPE could just have their own Kafka and save the integration trouble. I’ll bet dollars to donuts that any Kafka provider would be more reliable too.
  * If we plan to push additional metrics through the orchestrator rather than the gateway, then I suspect a pull-based architecture would work better for us (Prometheus style) rather than have the orchs report their stats to a specially blessed metrics service, which is a form of centralization in itself. This also allows entities to build their own independent data pipelines (eg, Inc and Cloud) without needing to configure orchestrators for specific push-based collection methods such as Kafka or streamr or whatever.


  * John (elite encoder) &  Peter (Interptr)  wants to know if the AI Runner will have a package to allow any BYOC to leverage a consistant pckage for publishing measurments to StreamR

  * Brad wants to understand what "Failure Rate" means as a metric. He also feels it should come from the gateway unless its a hardware failure.

  * Brad expresses concerns about "synthetic workloads" and the impact it has on production workloads. 
    * Qiang belives orchs should have some sorta queue of jobs and tests always are lower prioroity over production workloads

  * Qiang is saying kafka and clickhouse will be used until after milestone 1 adn wait till Streamr is proven, then migrate from kafka to the Streamr approach

  * Rafal doesnt understand what a synthetic workload is
    * Qiang says "he audit gateway, needs to present a more close-2-production traffic, to obtain the objetive metrics of the whole network. 

      to do this, the workload needs to be generated, not just a set of stale/unchanging set. for example, prompt as one most important traffic input, it can be generated using a rule, to simulate the diversity of prompt, without being too biased. it is a widely used approach in model training and tuning field. we should leverage this approach, to ensure, we do not just use a constant set of prompts, for example, for all models, and workloads. 

      good prompt from krea model, is a good example, they generate different size of prompts from a set of core basic prompts, to ensure their models are performing as expected. we can certainly learn some from that, to earn the crediability from the community at large, our network is performing as we said performant. "

* Brad doesnt think AI Runner should publish metrics to streamr, he thinks it should bubble up to orch and get published.

* Qiang says "It is important to have an evolving test dataset that can capture the evolvement of"
  * Brad says "I think a framework for submitting tests want ran on the network makes sense.  The start stream requests are relatively similar with main items being height, width and params. These values coupled with the pipeline/model_id to run it with creates what and how to run for the dataset."


* Rafal doesnt understand a "Shared Datset" as it relates to synthetic workloads
  * How will this dataset be created? Each O can server different model, so the output data may not be comparable?
  * He advises: What we should do is to for each O:
    1. Get what the “primary” (warm) model O support
    2. Test the given O with that model
    3. Record the data but classify it with the specific model
    4. Use only this for the selection

* Qiang reiterates: "NOTE: for all telemetry events that define a metrics/measurements, the following id must be present to ensure the completeness of the event"
  * what is MUST-HAVE metadata to identify it??
  * Rafal states "Yes, I think we need to include the ID in all the telemetry data."
    
    * The difficulty here is that AFAIU we have a few different stream IDs (and we should include all of them):
      - stream Name ⇒ aka. stream key, something that users know
      - stream ID ⇒ randomly generated ID of the stream created when the user starts streaming
      - request ID ⇒ randomly generated ID for each user streaming retry
      - trickle ID ⇒ internal manifest ID used the the G<>O<>R communication

* Qiang states " this is the extended scope that can be optional for milestone 1. however, when you create the first dashboard, many will be surfacing, as those APIs are backbone I would assume, for a dashboard, or any data viz to be built with a well-defined data interface, at its foundation. "
    *  "Gateway implements a set of APIs that allow SLAs related metrics to be queried"

* Qiang wants to know if the following is accurate:
    *  "│ Gateway reports pathway metrics such as (e2e latency, swap rate) directly to “streamr” infra"

    