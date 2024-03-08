+++
title = 'Managing the Lifecycle of your Elasticsearch Indices'
description = 'How to efficiently manage index lifecycles in an automated and clean manner'
date = 2020-05-13T17:59:22+01:00
draft = true
+++
## Managing the Lifecycle of your Elasticsearch Indices

__Just like me, you are probably__ storing your __[Applications | Infrastructure | IoT ] Logs / Traces__ ([as a time series](https://en.wikipedia.org/wiki/Time_series)) into Elasticsearch or at least __considering doing it.__

__If that is the case, you might be wondering how to efficiently manage index lifecycles in an automated and clean manner, then this post is for you!__

### What's happening?

Basically, this means that your log management/aggregator applications are storing the logs in Elasticsearch using the __timestamp__ (of capture, processing, or another one) for every record of data and __grouping, using a pattern for every group.__

In Elasticsearch terms, this __group of logs__ is called index and the pattern is referring commonly to the suffix used when you create the index name, e.g.: __sample-logs-2020-04-25.__

### The problem

Until here everything is ok, right? so the problems begin when your data starts accumulating and you don't want to spend too much time/money to store/maintain/delete it.

Additionally, you may be managing all the indices the same, regardless of data retention requirements or access patterns. All the indices have the same number of replicas, shards, disk type, etc. In my case, it is more important the first week of indices than the indices three months old.

As I mentioned before, depending on your index name and configuration, you will end up with different indexes aggregating logs based on different timeframes.

```text
...
sample-logs-2020-04-22
sample-logs-2020-04-23
sample-logs-2020-04-24
...
sample-logs-2020-04-27
```

It is likely that just like I was doing few months back, you are using your custom Script/Application implementing the [Elasticsearch Curator API](https://www.elastic.co/guide/en/elasticsearch/client/curator/current/index.html) or going directly over the [Elasticsearch index API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html) to delete or maintain your indices lifecycle or worst, you are storing your indices forever without any kind of control or deleting it manually.

### My logs cases

#### Case 1

Third-party applications that use their own index name pattern like __indexname-yyyy-mm-dd__ and I cannot / I don't want to change it.  e.g. [Zipkin](https://zipkin.io/) (zipkin-2020-04-25)

![image 1](</blog/post/2020/05/13/1/images/logs-open-distro-elasticsearch-1.jpg>)

#### Case 2

My own log aggregator (custom [AWS Lambda function](https://aws.amazon.com/lambda)) and/or third-party applications like fluentd, LogStash,  etc. That allows me to change the index name pattern. So, in this case I can decide how to aggregate my logs and the index pattern name I want to use.

![image 1](</blog/post/2020/05/13/1/images/logs-open-distro-elasticsearch-2.jpg>)

>__One more thing before moving on__: The term used for Elasticsearch to create a index per day, hour, month, etc. is __rollover.__

### The Solution

#### Preliminaries

In my case, I’m using [AWS Elasticsearch Service](https://aws.amazon.com/elasticsearch-service) which is a [“little bit different”](https://logz.io/blog/open-distro-for-elasticsearch/) from the [Elasticsearch Elastic](https://www.elastic.co/elasticsearch/) since [AWS decided to create their own Elasticsearch fork](https://aws.amazon.com/blogs/aws/new-open-distro-for-elasticsearch/) called [Open Distro for Elasticsearch](https://opendistro.github.io/for-elasticsearch/).

the key terms to understand __“Index Lifecycle”__ in every Elasticsearch distribution is:

+ [Index State Management (ISM)](https://opendistro.github.io/for-elasticsearch-docs/docs/ism/)  → Open Distro for Elasticsearch
+ [Index lifecycle management (ILM)](https://www.elastic.co/guide/en/elasticsearch/reference/current/overview-index-lifecycle-management.html) → Elasticsearch Elastic

ElasticSearch concepts are out of the scope of this post, in the below cases I will explain how __Open Distro for Elasticsearch__ manages its indices lifecycle.

#### Case 1

... Remember above.

> The log management/aggregation application makes the “rollover” of my indices, but I would like to delete/change those after the index has rolled — The most common


Create an [Index State Management Policy](https://opendistro.github.io/for-elasticsearch-docs/docs/ism/policies/) to delete indices __based on time and/or size__ and using an [Elasticsearch Templates](https://opendistro.github.io/for-elasticsearch-docs/docs/elasticsearch/index-templates/) and [Elasticsearch Aliases](https://opendistro.github.io/for-elasticsearch-docs/docs/elasticsearch/index-alias/) your Elasticsearch engine can delete your indices periodically.

>And yes, the result is very similar to what I was doing with my custom AWS Lambda function in Python using Elasticsearch Curator API

> But, without the hassle of writing any code, handling connection errors, upgrade my code every time my Elasticsearch was upgraded, credentials, changing the env vars to pass the new indices name, etc.

Now, thanks to __ISM__ I can use a __JSON declarative language__ to define some rules __(policies)__ and the Elasticsearch engine is in charge of the rest.

__Policies?__ … imagine you can implement this kind of rules

1. __Keep__ “my fresh indices” open to write for __2 days__, then
2. After the 2 first days, closes those indices for write operations and keep them __until 13 days__ more, then
3. __15 days__ after index creation, __delete it forever__, end

__What does this means?__ well, after learning about the [ISM Policies](https://opendistro.github.io/for-elasticsearch-docs/docs/ism/policies/) and using[ Kibana Dev Tool](https://github.com/christiangda/es-lifecycle-ism/blob/master/images/kibana-dev-tool.png), I created a policy name __delete_after_15d__ following the rules described above, and here you have it.

```bash {linenos=table,hl_lines=[2,5,14,21,30,37,46]}
# ISM Policy delete_after_15d
PUT _opendistro/_ism/policies/delete_after_15d
{
    "policy": {
        "policy_id": "delete_after_15d",
        "description": "Maintains the indices open by 2 days, then closes those and delete indices after 15 days",
        "default_state": "ReadWrite",
        "schema_version": 1,
        "states": [
            {
                "name": "ReadWrite",
                "actions": [
                    {
                        "read_write": {}
                    }
                ],
                "transitions": [
                    {
                        "state_name": "ReadOnly",
                        "conditions": {
                            "min_index_age": "2d"
                        }
                    }
                ]
            },
            {
                "name": "ReadOnly",
                "actions": [
                    {
                        "read_only": {}
                    }
                ],
                "transitions": [
                    {
                        "state_name": "Delete",
                        "conditions": {
                            "min_index_age": "13d"
                        }
                    }
                ]
            },
            {
                "name": "Delete",
                "actions": [
                    {
                        "delete": {}
                    }
                ]
            }
        ]
    }
}
```

__NOTE:__ Notice the highlighted lines, are these my rules described above?

Then using the following [Elasticsearch Templates](https://opendistro.github.io/for-elasticsearch-docs/docs/elasticsearch/index-templates/), I applied the policy (above), see template line (below) 8, to my indices following a pattern in the line 5

```bash {linenos=table,hl_lines=[2,5,8]}
# Template sample-logs to apply the ISM Policy delete_after_15d to new indices
PUT _template/sample-logs
{
    "index_patterns": [
        "sample-logs-*"
    ],
    "settings": {
        "index.opendistro.index_state_management.policy_id": "delete_after_15d"
    }
}
```

__Now what? Is it ready?__

For the __new indices__, yes. The indices created after you created this template into your Elasticsearch.

__What about the old ones?__

The indices created before applying the index template. For these we need to change its definition and add the line 5

```bash {linenos=table,hl_lines=[5]}
# Change the oldest indices definition to apply the ISM Policy delete_after_15d
PUT sample-logs-2020-*/_settings
{
  "settings": {
    "index.opendistro.index_state_management.policy_id": "delete_after_15d"
  }
}
```

__But, how do I complete the tasks you mention before?__

Don’t worry, keep calm!, [here https://github.com/slashdevops/es-lifecycle-ism](https://github.com/slashdevops/es-lifecycle-ism) you have the complete explanation to apply this rule in your own Elasticsearch, also how to test it into an Elasticsearch instance or create it locally with docker-compose.

#### Case 2

> My Elasticsearch rolls over the indices base on time and/or size and I want to have only one entry point (index) to send my logs — I think it is the best one

The rules again

1. Rollover __“my fresh indices”__ after 1 day, then
2. Close those indices for write operations and keep it __until 13 days more__, then
3. __After 15 days__ of the index was created, __delete it forever__, end

Well, to do that I create an __ISM policy__ named __rollover_1d_delete_after_15__ to control the state of my indices and using the rollover action.

```bash {linenos=table,hl_lines=[2,5,14,15,16,23,39,48]}
# ISM Policy rollover_1d_delete_after_15
PUT _opendistro/_ism/policies/rollover_1d_delete_after_15
{
    "policy": {
        "policy_id": "rollover_1d_delete_after_15",
        "description": "Rollover every 1d, then closes those and delete indices after 15 days",
        "default_state": "Rollover",
        "schema_version": 1,
        "states": [
            {
                "name": "Rollover",
                "actions": [
                    {
                        "rollover": {
                            "min_index_age": "1d"
                        }
                    }
                ],
                "transitions": [
                    {
                        "state_name": "ReadOnly",
                        "conditions": {
                            "min_index_age": "2d"
                        }
                    }
                ]
            },
            {
                "name": "ReadOnly",
                "actions": [
                    {
                        "read_only": {}
                    }
                ],
                "transitions": [
                    {
                        "state_name": "Delete",
                        "conditions": {
                            "min_index_age": "13d"
                        }
                    }
                ]
            },
            {
                "name": "Delete",
                "actions": [
                    {
                        "delete": {}
                    }
                ],
                "transitions": []
            }
        ]
    }
}
```

__NOTE:__ Notice the highlighted lines, did you see the rollover action?

Then [like in case 1](#case-1), using the following [Elasticsearch Templates](https://opendistro.github.io/for-elasticsearch-docs/docs/elasticsearch/index-templates/), I applied the policy above, see template (below) line 7, to my indices following a pattern in the line 5.

```bash {linenos=table,hl_lines=[2,5,7,8,9]}
# Template sample-logs-rollover to apply the ISM Policy
# rollover_1d_delete_after_15 to new indices
PUT _template/sample-logs-rollover
{
    "index_patterns": [
        "sample-logs-rollover-*"
    ],
    "settings": {
        "index.opendistro.index_state_management.policy_id": "rollover_1d_delete_after_15",
        "index.opendistro.index_state_management.rollover_alias": "sample-logs-rollover"
    }
}
```

__What does it means?__

Now Elasticsearch Engine will be in charge of rollover the indices and you don’t need to create any index name pattern when indexing your data over Elasticsearch, in other words, your Application logs’ aggregator doesn’t need to rollover your indices.

The last step and obligatory to trigger all the rollover processes inside Elasticsearch, it creates the first rollover index according to the template and aliases defined inside this.

```bash {linenos=table,hl_lines=[2,5,6]}
# Create the first rollover manually (it is necessary)
# to trigger ISM Policy association
PUT sample-logs-rollover-000001
{
    "aliases": {
        "sample-logs-rollover":{
            "is_write_index": true
        }
    }
}
```

__So, How do I index my data now?__

Using the rollover alias (template definition above line 5) created in the Elasticsearch template. Now you have only one index name (index alias) to configure your Custom Program / LogStash / Fluentd, etc and you can forget the suffix pattern.

Here is an example of how to insert data __using the rollover index alias__:

```bash {linenos=table,hl_lines=[2,3,5]}
# Bulk load sample, NOTE: To insert data use the rollover aliases
POST _bulk
{"index": { "_index": "sample-logs-rollover"}}
{"message": "This is a log sample 1", "@timestamp": "2020-04-26T11:07:00+0000"}
{"index": { "_index": "sample-logs-rollover"}}
{"message": "This is a log sample 2", "@timestamp": "2020-04-26T11:08:00+0000"}
```

### Conclusions

If you have Elasticsearch as your logs storage and index platform and you never used before or heard about it:

+ [Index State Management (ISM)](https://opendistro.github.io/for-elasticsearch-docs/docs/ism/)  → Open Distro for Elasticsearch
+ [Index lifecycle management (ILM)](https://www.elastic.co/guide/en/elasticsearch/reference/current/overview-index-lifecycle-management.html) → Elasticsearch Elastic

Then, Go fast and learn how to apply this to improve your everyday job.

### Acknowledgements

This was possible thanks to my friend __Alejandro Sabater__, who took his free time to review it and share its recommendation with me.
