---
layout: default
title: Composite monitors
nav_order: 3
parent: Alerting
has_children: false
redirect_from:
---

# Composite monitors

---

#### Table of contents
- TOC
{:toc}

---

## About composite monitors

Basic [monitor types]({{site.url}}{{site.baseurl}}/observing-your-data/alerting/monitors/#monitor-types) for the Alerting plugin are designed to define a single trigger type. For example, a document level monitor can trigger an alert based on a query's match with documents, while a bucket level monitor can trigger an alert based on queries aimed at aggregated values in a data source. The composite monitor extends the usefulness of the individual monitor types by providing functionality that can chain the output of multiple monitor types into a single workflow that includes multiple triggers and permits analysis of a data source based on multiple criteria. This allows you to derive more granular information about a data source, and it doesn't require you to manually coordinate the scheduling of the separate monitors.

Composite monitors solve limitations with basic monitors in the following ways:

* Composite monitors give you the ability to create complex queries through a combination of triggers generated by multiple types of monitors.
* They have the capacity to define a pipeline of rules and queries that are run as a single execution.
* They can chain together individual monitors so that the output of one can act as the input for the next, repeating this through the entire sequence of the execution.
* They provide a more complete view of a given data source by running multiple monitors and multiple types of monitors in sequence, creating finer focused results and reducing noise in the results.


## Key terms

The key terms in the following table describe the basic concepts behind composite monitors. For additional terms common to all types of monitors, see [Key terms]({{site.url}}{{site.baseurl}}/observing-your-data/alerting/monitors/#key-terms) for basic monitors.

| Term | Definition |
| :--- | :--- |
| Composite monitor | A combination of multiple individual monitors that includes functionality to execute each monitor in a sequence. The compound monitor can query different aspects of a dataset based on the type of monitors included in the composite monitor's definition.  |
| Delegate monitor | Any monitor of any monitor type used in the chained workflow of a compound monitor. Delegate monitors are executed sequentially according to their order in the monitor definition. |
| Findings chain | A sequence of findings where the findings for each monitor are used as the inputs for subsequent monitors. 
| Alert chain | A sequence of alerts where an alert for each monitor is used as the input for a subsequent monitor.  |
| Execution | A single run of all delegate monitors in the sequence defined in the composite monitor's configuration. |
| Execution Id | Allows for the management of data recorded from a specific execution of a composite monitor. The execution Id associates findings and alerts with the execution and is stored in each monitor's metadata, along with the workflow Id and the monitor Id. |


## Example workflows

In a composite monitor, the chained outputs from the individual delegate monitors can be either findings or alerts. The following sections describe the workflow for each and provide an example of how they work.


### Chained findings

As an example of chained findings, consider a composite monitor configured with two delegate monitors. The second monitor is defined to use findings from the first as its input.

* The first delegate monitor (monitor #1) is a per document monitor configured to analyze a data source with the following three queries:
   
   1. Request payload size greater than 100 Kb.
   2. Response status = 200.
   3. Response contains a specific header.
   
   For every execution, the monitor queries the data source and generates findings for documents that match the conditions set out by the queries. These results by themselves are valuable. However, by adding a second monitor the sequence can apply further analysis to events in the data source.
* The second monitor (monitor #2) is a per bucket monitor that aggregates data by client IP, checks how many IPs there are, and determines how many IPs are sending these types of requests. The composite monitor configuration provides a way to add the second monitor in sequence so that it executes following the first.
* Monitor #2 first filters the findings from Monitor #1 and then queries the data derived from monitor #1. Matches with this second set of queries then generate triggers and alerts for notifications.
* Both monitors #1 and #2 return trigger results to the composite monitor according to their sequence in the configuration. The composite monitor returns a list of trigger results from monitors #1 and #2.

The following image shows a simplified workflow for a composite monitor with chained findings.

{% include gif-pause.html %}


### Chained alerts

As an example of chained alerts, consider a composite monitor configured with three delegate monitors. The first and second monitors (monitors #1 and #2) are configured to generate alerts when two specific events happen across the cluster. A third monitor (monitor #3) is configured to send an alert when both monitors #1 and #2 generate their own alerts.

* Delegate monitor #1 is configured to monitor CPU utilization of a service’s worker nodes. The monitor includes trigger conditions that create alerts when the CPU experiences loads above a set threshold.
* Delegate monitor #2 is configured to monitor incoming request counts during the same time window. The monitor includes trigger conditions that create alerts when the number of requests rises above a set threshold.
* Delegate monitor #3 is configured to check whether monitors #1 and #2 have created alerts. If both monitors #1 and #2 have created alerts, monitor #3 creates its own alert and sends a notification that the service is experiencing a high volume of traffic and its performance is degraded.
* The first monitor's configuration may allow for alerts due to a number of reasons, such as cluster instability or several background processes running simultaneously. So an alert for this condition alone has limited value. Similarly, monitor #2 may identify a large number of requests and send an alert, even if the cluster is able to handle the traffic. Monitor #3 filters for these two conditions and triggers an alert when they both exist. This narrows the criteria for sending notifications, which also improves the meaningfulness of the alert while removing extraneous alerts that provide no deterministic value.

The following image shows a simplified workflow for a compound monitor with chained alerts.

{% include gif-pause2.html %}


## Managing composite monitors with the API

You can manage composite monitors using the REST API or OpenSearch Dashboards. Currently, the API offers the most versatility for defining composite monitors. The API configuration includes options to create composite monitors that chain findings and monitors that chain alerts. OpenSearch Dashboards is available for creating composite monitors that chain alerts.


### Create composite monitor

This API allows you to create composite monitors.

```json
POST _plugins/_alerting/workflows
```
{% include copy-curl.html %}


#### Request fields

| Field | Type | Description |
| :--- | :--- | :--- |
| `schedule` | Object | The schedule that determines how often the execution runs. |
| `schedule.period.interval` | Numeral | Accepts a numerical value to set how often the execution runs. |
| `schedule.period.unit` | Object | The time unit of measure for the interval. `SECONDS`, `MINUTES`, `HOURS`, `DAYS`. |
| `inputs` | Object | Accepts inputs to define the delegate monitors, which specify both the delegate monitors and their order in the execution's sequence. |
| `inputs.composite_input.sequence.delegates` | Object | Settings for the individual monitors that underlie the composite monitor. |
| `inputs.composite_input.sequence.delegates.order` | Number | Designates the order in which the monitor runs in the execution. |
| `inputs.composite_input.sequence.delegates.monitor_id` | String | The unique identifier for the monitor. |
| `enabled_time` | Number | TBD |
| `enabled` | Boolean | Setting to determine whether the composite monitor is enabled or not. Setting it to `true` enables the composite monitor. Default is TBD. |
| `workflow_type` | String | Set to `composite` for composite monitor. |
| `schema_version` | Number | TBD |
| `triggers` | Object | Details for the individual alert triggers. |
| `triggers.chained_alert_trigger` | Object | Details for each individual alert trigger. Each monitor's alert trigger will require settings for its configuration. |
| `triggers.chained_alert_trigger.id` | String | The unique identifier for the alert trigger. |
| `triggers.chained_alert_trigger.name` | String | TBD |
| `triggers.chained_alert_trigger.severity` | Number | The alert severity. 1 = highest; 2 = high; 3 = medium; 4 = low; 5 = lowest.|
| `triggers.chained_alert_trigger.condition.script` | O | The script details that determine the conditions for triggering an alert. |
| `triggers.chained_alert_trigger.condition.script.source` | String | The Painless script that defines the conditions for triggering an alert. |
| `triggers.chained_alert_trigger.condition.script.lang` | String | Enter `painless` for the Painless scripting language. |


#### Using Painless scripting language to define alert chains

Composite monitor configuration employs the [Painless scripting language](https://www.elastic.co/guide/en/elasticsearch/painless/7.17/painless-guide.html) to define the conditions for chaining alerts. Conditions are applied for each execution of the composite monitor. You specify the definition for the chain in the `triggers.chained_alert_triggers.condition.script.source` field of the request. Using Painless syntax, you can apply logic to links between monitors with basic Boolean operators AND, OR, NOT, and precedence.

* AND = `&&`
* OR = `||`
* NOT = `!`
* Precedence = `()`

See the following examples to understand how each is used in the monitor definition.

* **Example 1**
   
   `monitor[id=1] && monitor[id=2]`
   
   The following conditions for delegate monitors will trigger the composite monitor to produce an alert when both monitor #1 AND monitor #2 generate an alert.

* **Example 2**
   
   `monitor[id=1] || monitor[id=2]`

   The following conditions will trigger the composite monitor to produce an alert when either monitor #1 OR monitor #2 generates an alert.

* **Example 3**
   
   `monitor[id=1] && (!monitor[id=2] || monitor[id=3])`

   The following conditions will trigger the composite monitor to produce an alert when monitor #1 generates an alert AND monitor #2 does NOT, OR, monitor #3 does generate an alert.
   
The order of monitor IDs in the Painless script does not define the sequence of execution for the monitors. The sequence of monitor execution is defined in the `inputs.composite_input.sequence.delegates.order` field in the request.
{: .note }


#### Example request

```json
POST _plugins/_alerting/workflows
{
    "last_update_time": 1679468231835,
    "owner": "alerting",
    "type": "workflow",
    "schedule": {
        "period": {
            "interval": 1,
            "unit": "MINUTES"
        }
    },
    "inputs": [
        {
            "composite_input": {
                "sequence": {
                    "delegates": [
                        {
                            "order": 1,
                            "monitor_id": "grsbCIcBvEHfkjWFeCqb"
                        },
                        {
                            "order": 2,
                            "monitor_id": "agasbCIcBvEHfkjWFeCqa"
                        }
                    ]
                }
            }
        }
    ],
    "enabled_time": 1679468231835,
    "enabled": true,
    "workflow_type": "composite",
    "schema_version": 0,
    "name": "scale_up",
    "triggers": [
        {
            "chained_alert_trigger": {
                "id": "m1ANDm2",
                "name": "jnkjn",
                "severity": "1",
                "condition": {
                    "script": {
                        "source": "(monitor[id={{m1}}] && monitor[id={{m2}}])", //ALERT WILL BE CREATED IF MONITOR 1 GENERATES ALERTS AND MONITOR 2 DOESN'T GENERATE ALERT",
                        "lang": "painless"
                    }
                }
            }
        },
        {
            "chained_alert_trigger": {
                "id": "m1ORm2",
                "name": "jnkjn",
                "severity": "1",
                "condition": {
                    "script": {
                        "source": "(monitor[id={{m1}}] || monitor[id={{m2}}])", //ALERT WILL BE CREATED IF MONITOR 1 GENERATES ALERTS OR MONITOR 2 GENERATE ALERT",
                        "lang": "painless"
                    }
                }
            }
        }
    ]
}
```
{% include copy-curl.html %}


### Get composite monitor

Retrieves information on the specified monitor.

```json
GET _plugins/_alerting/workflows/<id>
```
{% include copy-curl.html %}

#### Path parameters

| Field | Type | Description |
| :--- | :--- | :--- |
| monitor Id | String | The monitor Id |


### Update composite monitor

Updates the composite monitor's details. See [Create composite monitor](#create-composite-monitor) for description of request fields.

```json
PUT _plugins/_alerting/workflows/<id>
{
    "owner": "security_analytics",
    "type": "workflow",
    "schedule": {
        "period": {
        "interval": 1,
        "unit": "MINUTES"
        }
    },
    "inputs": [
        {
            "composite_input": {
                "sequence": {
                    "delegates": [
                        {
                            "order": 1,
                            "monitor_id": "grsbCIcBvEHfkjWFeCqb"
                        },
                        {
                            "order": 1,
                            "monitor_id": "agasbCIcBvEHfkjWFeCqa"
                        }
                    ]
                }
            }
        }
    ],
    "enabled_time": 1679468231835,
    "enabled": true,
    "workflow_type": "composite",
 
    "name": "NTxdwApKbv"
}
```
{% include copy-curl.html %}


### Delete composite monitor

```json
DELETE _plugins/_alerting/workflows/<id>
```
{% include copy-curl.html %}

#### Path parameters

| Field | Type | Description |
| :--- | :--- | :--- |
| monitor Id | String | The monitor Id |


### Execute composite monitor

```json
POST /_plugins/_alerting/workflows/<workflow_id>/_execute
```
{% include copy-curl.html %}

#### Path parameters

| Field | Type | Description |
| :--- | :--- | :--- |
| workflow Id | String | The workflow Id. Enter the workflow Id in the path to run the execution. |


#### Example response

```json
{
    "execution_id": "I0GXeIgBYKBG2nHoiHCL_2023-06-01T20:18:48.511884_a9c1d055-9b70-49c2-b32a-716cff1f562e",
    "workflow_name": "scale_up",
    "workflow_id": "I0GXeIgBYKBG2nHoiHCL",
    "trigger_results": {
        "m1ANDm2": {
            "name": "jnkjn",
            "triggered": true,
            "action_results": {},
            "error": null
        },
        "m1ORm2": {
            "name": "jnkjn",
            "triggered": true,
            "action_results": {},
            "error": null
        }
    },
    "monitor_run_results": [{
            "monitor_name": "test triggers",
            "period_start": 1685650668501,
            "period_end": 1685650728501,
            "error": null,
            "input_results": {
                "results": [{
                    "bhjh": [
                        "OkGceIgBYKBG2nHoyHAn|test1",
                        "O0GceIgBYKBG2nHozHCW|test1"
                    ],
                    "nkjkj": [
                        "OkGceIgBYKBG2nHoyHAn|test1",
                        "O0GceIgBYKBG2nHozHCW|test1"
                    ],
                    "jknkjn": [
                        "OkGceIgBYKBG2nHoyHAn|test1",
                        "O0GceIgBYKBG2nHozHCW|test1"
                    ]
                }],
                "error": null
            },
            "trigger_results": {
                "NC3Dd4cBCDCIfBYtViLI": {
                    "name": "njkkj",
                    "triggeredDocs": [
                        "OkGceIgBYKBG2nHoyHAn|test1",
                        "O0GceIgBYKBG2nHozHCW|test1"
                    ],
                    "action_results": {},
                    "error": null
                }
            }
        },
        {
            "monitor_name": "test triggers 2",
            "period_start": 1685650668501,
            "period_end": 1685650728501,
            "error": null,
            "input_results": {
                "results": [{
                    "bhjh": [
                        "PEGceIgBYKBG2nHo1HCw|test",
                        "PUGceIgBYKBG2nHo3HA8|test"
                    ],
                    "nkjkj": [
                        "PEGceIgBYKBG2nHo1HCw|test",
                        "PUGceIgBYKBG2nHo3HA8|test"
                    ],
                    "jknkjn": [
                        "PEGceIgBYKBG2nHo1HCw|test",
                        "PUGceIgBYKBG2nHo3HA8|test"
                    ]
                }],
                "error": null
            },
            "trigger_results": {
                "NC3Dd4cBCDCIfBYtViLI": {
                    "name": "njkkj",
                    "triggeredDocs": [
                        "PEGceIgBYKBG2nHo1HCw|test",
                        "PUGceIgBYKBG2nHo3HA8|test"
                    ],
                    "action_results": {},
                    "error": null
                }
            }
        }
    ],
    "execution_start_time": "2023-06-01T20:18:48.511874Z",
    "execution_end_time": "2023-06-01T20:18:53.682405Z",
    "error": null
}
```


### Acknowledge chained alerts

```json
POST _plugins/_alerting/workflows/<workflow-id>/_acknowledge/alerts
{
    "alerts": ["eQURa3gBKo1jAh6qUo49"]
}
```
{% include copy-curl.html %}

#### Path parameters

| Field | Type | Description |
| :--- | :--- | :--- |
| workflow Id | String | The workflow Id. Enter the workflow Id in the path to retrieve its associated alerts. |

#### Request fields

| Field | Type | Description |
| :--- | :--- | :--- |
| `alerts` | Array | A list of alerts by Id. The results include alerts that are acknowledged by the system as well as alerts not recognized by the system.  |

#### Example response

```json
{
    "success": [
    "eQURa3gBKo1jAh6qUo49"
    ],
    "failed": []
}
```

## Managing composite monitors in OpenSearch Dashboards

You can manage composite monitors in OpenSearch Dashboards and configure monitors that chain findings.
[waiting for UX completion and environment]

