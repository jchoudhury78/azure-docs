---
title: Indexing a Cosmos DB data source for Azure Search | Microsoft Docs
description: This article shows you how to create an Azure Search indexer with Cosmos DB as a data source.
services: search
documentationcenter: ''
author: chaosrealm
manager: pablocas
editor: 

ms.assetid: 
ms.service: search
ms.devlang: rest-api
ms.topic: article
ms.tgt_pltfrm: NA
ms.workload: search
ms.date: 08/10/2017
ms.author: eugenesh

---
# Connecting Cosmos DB with Azure Search using indexers

If you want to implement a great search experience over your Cosmos DB data, you can use an Azure Search indexer to pull data into an Azure Search index. In this article, we show you how to integrate Azure Cosmos DB with Azure Search without having to write any code to maintain indexing infrastructure.

To set up a Cosmos DB indexer, you must have an [Azure Search service](search-create-service-portal.md), and create an index, datasource, and finally the indexer. You can create these objects using the [portal](search-import-data-portal.md), [.NET SDK](/dotnet/api/microsoft.azure.search), or [REST API](/rest/api/searchservice/) for all non-.NET languages. 

If you opt for the portal, the [Import data wizard](search-import-data-portal.md) guides you through the creation of all these resources.

> [!NOTE]
> Azure Cosmos DB is the next generation of DocumentDB. Although the product name is changed, syntax is the same as before. Please continue to specify `documentdb` as directed in this indexer article. 

> [!TIP]
> You can launch the **Import data** wizard from the Cosmos DB dashboard to simplify indexing for that data source. In left-navigation, go to **Collections** > **Add Azure Search** to get started.

<a name="Concepts"></a>
## Azure Search indexer concepts
Azure Search supports the creation and management of data sources (including Cosmos DB) and indexers that operate against those data sources.

A **data source** specifies the data to index, credentials, and policies for identifying changes in the data (such as modified or deleted documents inside your collection). The data source is defined as an independent resource so that it can be used by multiple indexers.

An **indexer** describes how the data flows from your data source into a target search index. An indexer can be used to:

* Perform a one-time copy of the data to populate an index.
* Sync an index with changes in the data source on a schedule. The schedule is part of the indexer definition.
* Invoke on-demand updates to an index as needed.

<a name="CreateDataSource"></a>
## Step 1: Create a data source
To create a data source, do a POST:

    POST https://[service name].search.windows.net/datasources?api-version=2016-09-01
    Content-Type: application/json
    api-key: [Search service admin key]

	{
        "name": "mydocdbdatasource",
        "type": "documentdb",
        "credentials": {
            "connectionString": "AccountEndpoint=https://myDocDbEndpoint.documents.azure.com;AccountKey=myDocDbAuthKey;Database=myDocDbDatabaseId"
        },
        "container": { "name": "myDocDbCollectionId", "query": null },
        "dataChangeDetectionPolicy": {
            "@odata.type": "#Microsoft.Azure.Search.HighWaterMarkChangeDetectionPolicy",
            "highWaterMarkColumnName": "_ts"
        }
    }

The body of the request contains the data source definition, which should include the following fields:

* **name**: Choose any name to represent your Cosmos DB database.
* **type**: Must be `documentdb`.
* **credentials**:
  
  * **connectionString**: Required. Specify the connection info to your Azure Cosmos DB database in the following format: `AccountEndpoint=<Cosmos DB endpoint url>;AccountKey=<Cosmos DB auth key>;Database=<Cosmos DB database id>`
* **container**:
  
  * **name**: Required. Specify the id of the Cosmos DB collection to be indexed.
  * **query**: Optional. You can specify a query to flatten an arbitrary JSON document into a flat schema that Azure Search can index.
* **dataChangeDetectionPolicy**: Recommended. See [Indexing Changed Documents](#DataChangeDetectionPolicy) section.
* **dataDeletionDetectionPolicy**: Optional. See [Indexing Deleted Documents](#DataDeletionDetectionPolicy) section.

### Using queries to shape indexed data
You can specify a Cosmos DB query to flatten nested properties or arrays, project JSON properties, and filter the data to be indexed. 

Example document:

    {
        "userId": 10001,
        "contact": {
            "firstName": "andy",
            "lastName": "hoh"
        },
        "company": "microsoft",
        "tags": ["azure", "documentdb", "search"]
    }

Filter query:

    SELECT * FROM c WHERE c.company = "microsoft" and c._ts >= @HighWaterMark ORDER BY c._ts

Flattening query:

    SELECT c.id, c.userId, c.contact.firstName, c.contact.lastName, c.company, c._ts FROM c WHERE c._ts >= @HighWaterMark ORDER BY c._ts
    
    
Projection query:

    SELECT VALUE { "id":c.id, "Name":c.contact.firstName, "Company":c.company, "_ts":c._ts } FROM c WHERE c._ts >= @HighWaterMark ORDER BY c._ts


Array flattening query:

    SELECT c.id, c.userId, tag, c._ts FROM c JOIN tag IN c.tags WHERE c._ts >= @HighWaterMark ORDER BY c._ts

<a name="CreateIndex"></a>
## Step 2: Create an index
Create a target Azure Search index if you don’t have one already. You can create an index using the [Azure portal UI](search-create-index-portal.md), the [Create Index REST API](/rest/api/searchservice/create-index) or [Index class](/dotnet/api/microsoft.azure.search.models.index).

The following example creates an index with an id and description field:

    POST https://[service name].search.windows.net/indexes?api-version=2016-09-01
    Content-Type: application/json
    api-key: [Search service admin key]

	{
       "name": "mysearchindex",
       "fields": [{
         "name": "id",
         "type": "Edm.String",
         "key": true,
         "searchable": false
       }, {
         "name": "description",
         "type": "Edm.String",
         "filterable": false,
         "sortable": false,
         "facetable": false,
         "suggestions": true
       }]
     }

Ensure that the schema of your target index is compatible with the schema of the source JSON documents or the output of your custom query projection.

> [!NOTE]
> For partitioned collections, the default document key is Cosmos DB's `_rid` property, which gets renamed to `rid` in Azure Search. Also, Cosmos DB's `_rid` values contain characters that are invalid in Azure Search keys. For this reason, the `_rid` values are Base64 encoded.
> 
> 

### Mapping between JSON Data Types and Azure Search Data Types
| JSON DATA TYPE | COMPATIBLE TARGET INDEX FIELD TYPES |
| --- | --- |
| Bool |Edm.Boolean, Edm.String |
| Numbers that look like integers |Edm.Int32, Edm.Int64, Edm.String |
| Numbers that look like floating-points |Edm.Double, Edm.String |
| String |Edm.String |
| Arrays of primitive types, for example ["a", "b", "c"] |Collection(Edm.String) |
| Strings that look like dates |Edm.DateTimeOffset, Edm.String |
| GeoJSON objects, for example { "type": "Point", "coordinates": [long, lat] } |Edm.GeographyPoint |
| Other JSON objects |N/A |

<a name="CreateIndexer"></a>
## Step 3: Create an indexer

Once the index and data source have been created, you're ready to create the indexer:

    POST https://[service name].search.windows.net/indexers?api-version=2016-09-01
    Content-Type: application/json
    api-key: [admin key]

    {
      "name" : "mydocdbindexer",
      "dataSourceName" : "mydocdbdatasource",
      "targetIndexName" : "mysearchindex",
      "schedule" : { "interval" : "PT2H" }
    }

This indexer runs every two hours (schedule interval is set to "PT2H"). To run an indexer every 30 minutes, set the interval to "PT30M". The shortest supported interval is 5 minutes. The schedule is optional - if omitted, an indexer runs only once when it's created. However, you can run an indexer on-demand at any time.   

For more details on the Create Indexer API, check out [Create Indexer](https://docs.microsoft.com/rest/api/searchservice/create-indexer).

<a id="RunIndexer"></a>
### Running indexer on-demand
In addition to running periodically on a schedule, an indexer can also be invoked on demand:

    POST https://[service name].search.windows.net/indexers/[indexer name]/run?api-version=2016-09-01
    api-key: [Search service admin key]

> [!NOTE]
> When Run API returns successfully, the indexer invocation has been scheduled, but the actual processing happens asynchronously. 

You can monitor the indexer status in the portal or using the Get Indexer Status API, which we describe next. 

<a name="GetIndexerStatus"></a>
### Getting indexer status
You can retrieve the status and execution history of an indexer:

    GET https://[service name].search.windows.net/indexers/[indexer name]/status?api-version=2016-09-01
    api-key: [Search service admin key]

The response contains overall indexer status, the last (or in-progress) indexer invocation, and the history of recent indexer invocations.

    {
        "status":"running",
        "lastResult": {
            "status":"success",
            "errorMessage":null,
            "startTime":"2014-11-26T03:37:18.853Z",
            "endTime":"2014-11-26T03:37:19.012Z",
            "errors":[],
            "itemsProcessed":11,
            "itemsFailed":0,
            "initialTrackingState":null,
            "finalTrackingState":null
         },
        "executionHistory":[ {
            "status":"success",
             "errorMessage":null,
            "startTime":"2014-11-26T03:37:18.853Z",
            "endTime":"2014-11-26T03:37:19.012Z",
            "errors":[],
            "itemsProcessed":11,
            "itemsFailed":0,
            "initialTrackingState":null,
            "finalTrackingState":null
        }]
    }

Execution history contains up to the 50 most recent completed executions, which are sorted in reverse chronological order (so the latest execution comes first in the response).

<a name="DataChangeDetectionPolicy"></a>
## Indexing changed documents
The purpose of a data change detection policy is to efficiently identify changed data items. Currently, the only supported policy is the `High Water Mark` policy using the `_ts` (timestamp) property provided by Cosmos DB, which is specified as follows:

    {
        "@odata.type" : "#Microsoft.Azure.Search.HighWaterMarkChangeDetectionPolicy",
        "highWaterMarkColumnName" : "_ts"
    }

Using this policy is highly recommended to ensure good indexer performance. 

If you are using a custom query, make sure that the `_ts` property is projected by the query.

<a name="IncrementalProgress"></a>
### Incremental progress and custom queries
Incremental progress during indexing ensures that if indexer execution is interrupted by transient failures or execution time limit, the indexer can pick up where it left off next time it runs, instead of having to re-index the entire collection from scratch. This is especially important when indexing large collections. 

To enable incremental progress when using a custom query, ensure that your query orders the results by the `_ts` column. This enables periodic check-pointing that Azure Search uses to provide incremental progress in the presence of failures.   

In some cases, even if your query contains an `ORDER BY [collection alias]._ts` clause, Azure Search may not infer that the query is ordered by the `_ts`. You can tell Azure Search that results are ordered by using the `assumeOrderByHighWaterMarkColumn` configuration property. To specify this hint, create or update your indexer as follows: 

	{
     ... other indexer definition properties
     "parameters" : {
            "configuration" : { "assumeOrderByHighWaterMarkColumn" : true } }
    } 

<a name="DataDeletionDetectionPolicy"></a>
## Indexing deleted documents
When rows are deleted from the collection, you normally want to delete those rows from the search index as well. The purpose of a data deletion detection policy is to efficiently identify deleted data items. Currently, the only supported policy is the `Soft Delete` policy (deletion is marked with a flag of some sort), which is specified as follows:

    {
        "@odata.type" : "#Microsoft.Azure.Search.SoftDeleteColumnDeletionDetectionPolicy",
        "softDeleteColumnName" : "the property that specifies whether a document was deleted",
        "softDeleteMarkerValue" : "the value that identifies a document as deleted"
    }

If you are using a custom query, make sure that the property referenced by `softDeleteColumnName` is projected by the query.

The following example creates a data source with a soft-deletion policy:

	POST https://[Search service name].search.windows.net/datasources?api-version=2016-09-01
    Content-Type: application/json
    api-key: [Search service admin key]

    {
        "name": "mydocdbdatasource",
        "type": "documentdb",
        "credentials": {
            "connectionString": "AccountEndpoint=https://myDocDbEndpoint.documents.azure.com;AccountKey=myDocDbAuthKey;Database=myDocDbDatabaseId"
        },
        "container": { "name": "myDocDbCollectionId" },
        "dataChangeDetectionPolicy": {
            "@odata.type": "#Microsoft.Azure.Search.HighWaterMarkChangeDetectionPolicy",
            "highWaterMarkColumnName": "_ts"
        },
        "dataDeletionDetectionPolicy": {
            "@odata.type": "#Microsoft.Azure.Search.SoftDeleteColumnDeletionDetectionPolicy",
            "softDeleteColumnName": "isDeleted",
            "softDeleteMarkerValue": "true"
        }
    }

## <a name="NextSteps"></a>Next steps
Congratulations! You have learned how to integrate Azure Cosmos DB with Azure Search using the indexer for Cosmos DB.

* To learn how more about Azure Cosmos DB, see the [Azure Cosmos DB service page](https://azure.microsoft.com/services/cosmos-db/).
* To learn how more about Azure Search, see the [Search service page](https://azure.microsoft.com/services/search/).
