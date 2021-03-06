---
title: Transform data with Databricks Notebook - Azure | Microsoft Docs
description: Learn how to process or transform data by running a Databricks notebook.
services: data-factory
documentationcenter: ''
author: douglaslMS
manager: craigg

ms.assetid: 
ms.service: data-factory
ms.workload: data-services
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 03/15/2018
ms.author: douglasl
---
# Transform data by running a Databricks notebook

The Azure Databricks Notebook Activity in a [Data Factory pipeline](concepts-pipelines-activities.md) runs a Databricks notebook in your Azure Databricks workspace. This article builds on the [data transformation activities](transform-data.md) article, which presents a general overview of data transformation and the supported transformation activities. Azure Databricks is a managed platform for running Apache Spark.

## Databricks Notebook activity definition

Here is the sample JSON definition of a Databricks Notebook Activity:

```json
{
	"activity": {
		"name": "MyActivity",
		"description": "MyActivity description",
		"type": "DatabricksNotebook",
		"linkedServiceName": {
			"referenceName": "MyDatabricksLinkedservice",
			"type": "LinkedServiceReference"
		},
		"typeProperties": {
			"notebookPath": "/Users/user@example.com/ScalaExampleNotebook",
			"baseParameters": {
				"inputpath": "input/folder1/",
				"outputpath": "output/"
			}
		}
	}
}
```

## Databricks Notebook activity properties

The following table describes the JSON properties used in the JSON
definition:

|Property|Description|Required|
|---|---|---|
|name|Name of the activity in the pipeline.|Yes|
|description|Text describing what the activity does.|No|
|type|For Databricks Notebook Activity, the activity type is DatabricksNotebook.|Yes|
|linkedServiceName|Name of the Databricks Linked Service on which the Databricks notebook runs. To learn about this linked service, see [Compute linked services](compute-linked-services.md) article.|Yes|
|notebookPath|The absolute path of the notebook to be run in the Databricks Workspace. This path must begin with a slash.|Yes|
|baseParameters|An array of Key-Value pairs. Base parameters can be used for each activity run. If the notebook takes a parameter that is not specified, the default value from the notebook will be used. Find more on parameters in [Databricks Notebooks](https://docs.databricks.com/api/latest/jobs.html#jobsparampair).|No|
