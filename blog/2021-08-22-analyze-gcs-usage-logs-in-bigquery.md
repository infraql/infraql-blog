---
slug: analyze-gcs-usage-logs-in-bigquery
title: Enable Logging for Google Cloud Storage Buckets and Analyzing Logs in Big Query (Part II)
author: Jeffrey Aven
author_title: Cloud Consultant
author_url: https://github.com/infraql
hide_table_of_contents: false
author_image_url: https://s.gravatar.com/avatar/f96573d092470c74be233e1dded5376f?s=80
image: https://storage.googleapis.com/infraql-web-assets/blog/infraql-gcs-logging-to-bq.png
description: This post demonstrates how to enable logging for a Google Cloud Storage bucket and analyze usage logs in Big Query using InfraQL - a new, SQL based approach to deploying and querying cloud resources.
keywords: [infraql, google cloud, GCP, infracoding, IaC, infrastructure as code, google cloud storage, cloud storage, GCS, logging, bigquery]
tags: [infraql, google cloud, GCP, infracoding, IaC, infrastructure as code, google cloud storage, cloud storage, GCS, logging, bigquery]
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

In the previous [__post__](/blog/enable-google-cloud-storage-logging), we showed you how to enable usage and storage logging for GCS buckets.  Now that we have enabled logging, let's load and analyze the logs using Big Query.  We will build up a data file __`vars.jsonnet`__ as we go and show the queries step by step, at the end we will show how to run this as one batch using InfraQL.  

## Step 1 : Create a Big Query dataset

We will need a dataset (akin to a schema or a database in other RDMBS parlance), basically a container for objects such as tables or views, the data and code to do this are shown here:  

<Tabs
  defaultValue="iql"
  values={[
    { label: 'InfraQL', value: 'iql', },
    { label: 'Data', value: 'data', },
  ]
}>
<TabItem value="iql">

```jsx
INSERT INTO google.bigquery.datasets(
  projectId,
  data__location,
  data__datasetReference,
  data__description,
  data__friendlyName
)
SELECT
  '{{ .projectId }}',
  '{{ .location }}',
  '{ "datasetId": "{{ .datasetId }}", "projectId": "{{ .projectId }}" }',
  '{{ .description }}',
  '{{ .friendlyName }}'
;
```
</TabItem>
<TabItem value="data">

```jsx
// variables
local projectId = 'infraql';
local datasetId = 'infraql_downloads';

// config
{
  projectId: projectId,
  datasetId: datasetId,
  location: 'US',
  description: 'Storage and usage logs from the website',
  friendlyName: 'InfraQL Download Logs',
}
```
</TabItem>
</Tabs>

## Step 2 : Create `usage` table

Let's use InfraQL to create a table named `usage` to host the GCS usage logs, the schema for the table is defined in a file named [`cloud_storage_usage_schema_v0.json`](http://storage.googleapis.com/pub/cloud_storage_usage_schema_v0.json) which can be downloaded from the location provided, for reference this is provided in the __Table Schema__ tab in the example provided below:  

<Tabs
  defaultValue="iql"
  values={[
    { label: 'InfraQL', value: 'iql', },
    { label: 'Data', value: 'data', },
    { label: 'Table Schema', value: 'schema', },
  ]
}>
<TabItem value="iql">

```jsx
INSERT INTO google.bigquery.tables(
  datasetId,
  projectId,
  data__description,
  data__friendlyName,
  data__tableReference,
  data__schema
)
SELECT
  '{{ .datasetId }}',
  '{{ .projectId }}',
  '{{ .table.usage.description }}',
  '{{ .table.usage.friendlyName }}',
  '{"projectId": "{{ .projectId }}", "datasetId": "{{ .datasetId }}", "tableId": "{{ .table.usage.tableId }}"}',
  '{{ .table.usage.schema }}'
; 
```
</TabItem>
<TabItem value="data">

```jsx
// variables
local projectId = 'infraql';
local datasetId = 'infraql_downloads';
local usage_fields = import 'cloud_storage_usage_schema_v0.json';

// config
{
  projectId: projectId,
  datasetId: datasetId,
  location: 'US',
  description: 'Storage and usage logs from the website',
  friendlyName: 'InfraQL Download Logs',
  table: {
    usage: {
      tableId: 'usage',
      friendlyName: 'Usage Logs',
	  description: 'Big Query table for GCS usage logs',
	  schema: {
        fields: usage_fields
	  }
    },
  },
}
```
</TabItem>

<TabItem value="schema">

```json
[
  {
    "name": "time_micros",
    "type": "integer",
    "mode": "NULLABLE"
  },

  {
    "name": "c_ip",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "c_ip_type",
    "type": "integer",
    "mode": "NULLABLE"
  },

  {
    "name": "c_ip_region",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_method",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_uri",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "sc_status",
    "type": "integer",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_bytes",
    "type": "integer",
    "mode": "NULLABLE"
  },

  {
    "name": "sc_bytes",
    "type": "integer",
    "mode": "NULLABLE"
  },

  {
    "name": "time_taken_micros",
    "type": "integer",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_host",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_referer",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_user_agent",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "s_request_id",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_operation",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_bucket",
    "type": "string",
    "mode": "NULLABLE"
  },

  {
    "name": "cs_object",
    "type": "string",
    "mode": "NULLABLE"
  }
]
```
</TabItem>
</Tabs>

## Step 3 : Load the usage data

We have a Big Query dataset and a table, lets load some data.  To do this we will need to create and submit a `load` job, we can do this by inserting into the `google.bigquery.jobs` resource as shown here:  

<Tabs
  defaultValue="iql"
  values={[
    { label: 'InfraQL', value: 'iql', },
    { label: 'Data', value: 'data', },
  ]
}>
<TabItem value="iql">

```jsx
INSERT INTO google.bigquery.jobs(
  projectId,
  data__configuration
)
SELECT
  'infraql',
  '{
    "load": {
      "destinationTable": {
        "projectId": "{{ .projectId }}",
        "datasetId": "{{ .datasetId }}",
        "tableId": "{{ .table.usage.tableId }}"
      },
      "sourceUris": [
        "gs://{{ .logs_bucket }}/{{ .object_prefix }}"
      ],
      "schema": {{ .table.usage.schema }},
	  "skipLeadingRows": 1,
      "maxBadRecords": 0,
      "projectionFields": []
    }
  }'
;
```
</TabItem>
<TabItem value="data">

```jsx
// variables
local projectId = 'infraql';
local datasetId = 'infraql_downloads';
local usage_fields = import 'cloud_storage_usage_schema_v0.json';

// config
{
  projectId: projectId,
  datasetId: datasetId,
  location: 'US',
  logs_bucket: 'infraql-download-logs',
  object_prefix: 'infraql_downloads_usage_2021*',
  description: 'Storage and usage logs from the website',
  friendlyName: 'InfraQL Download Logs',
  table: {
    usage: {
      tableId: 'usage',
      friendlyName: 'Usage Logs',
	  description: 'Big Query table for GCS usage logs',
	  schema: {
        fields: usage_fields
	  }
    },
  },
}
```
</TabItem>
</Tabs>

## Clean up (optional)

If you want to clean up what you have done, you can do so using InfraQL `DELETE` statements, as provided below:

> NOTE: To delete a Big Query dataset, you need to delete all of the tables contained in the dataset first, as shown in the following example

<Tabs
  defaultValue="iql"
  values={[
    { label: 'InfraQL', value: 'iql', },
    { label: 'Data', value: 'data', },
  ]
}>
<TabItem value="iql">

```jsx
-- delete table(s) 

DELETE FROM google.bigquery.tables 
WHERE projectId = '{{ .projectId }}' 
AND datasetId = '{{ .datasetId }}' 
AND tableId = '{{ .table.usage.tableId }}';

-- delete dataset

DELETE FROM google.bigquery.datasets 
WHERE projectId = '{{ .projectId }}' 
AND datasetId = '{{ .datasetId }}';
```
</TabItem>
<TabItem value="data">

```jsx
// generally you would use the same data used to create the dataset and table(s)  

// variables
local projectId = 'infraql';
local datasetId = 'infraql_downloads';

// config
{
  projectId: projectId,
  datasetId: datasetId,
  table: {
    usage: {
      tableId: 'usage',
    }
  },  
}
```
</TabItem>
</Tabs>