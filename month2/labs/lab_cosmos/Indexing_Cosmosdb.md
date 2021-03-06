# Azure Cosmos DB Lab
## Lab 3: Indexing in Azure Cosmos DB

In this lab, you will modify the indexing policy of an Azure Cosmos DB container. You will explore how you can optimize indexing policy for write or read heavy workloads as well as understand the indexing requirements for different SQL API query features.

## Indexing Overview

Azure Cosmos DB is a schema-agnostic database that allows you to iterate on your application without having to deal with schema or index management. By default, Azure Cosmos DB automatically indexes every property for all items in your container without the need to define any schema or configure secondary indexes. If you chose to leave indexing policy at the default settings, you can run most queries with optimal performance and never have to explicitly consider indexing. However, if you want control over adding or removing properties from the index, modification is possible through the Azure Portal or any SQL API SDK.

Azure Cosmos DB uses an inverted index, representing your data in a tree form. For a brief introduction on how this works, read our [indexing overview](https://docs.microsoft.com/en-us/azure/cosmos-db/index-overview) before continuing with the lab.

## Customizing the indexing policy

In this lab section, you will view and modify the indexing policy for your **FoodCollection**.

### Open Data Explorer

1. On the left side of the portal, select the **Resource groups** link.

2. In the **Resource groups** blade, locate and select the resource group created for this lab.

3. In the resource group, select your **Azure Cosmos DB** account.

4. In the **Azure Cosmos DB** blade, locate and select the **Data Explorer** link on the left side of the blade.

5. In the **Data Explorer** section, expand the **NutritionDatabase** database node and then expand the **FoodCollection** container node.

6. Within the **FoodCollection** node, select the **Items** link.

7. View the items within the container. Observe how these documents have many properties, including arrays. If we do not use a particular property in the WHERE clause, ORDER BY clause, or a JOIN, indexing the property does not provide any performance benefit.

8. Still within the **FoodCollection** node, select the **Scale & Settings** link.
9. Review the **Indexing Policy** section

    - Notice you can edit the JSON file that defines your container's index
    - An Indexing policy can also be modified through any Azure Cosmos DB SDK
    - During this lab we will modify the indexing policy through the Azure Portal

   ![The Indexing Policy window is highlighted](./images/lab03/03-indexingpolicy-initial.jpg "Review the indexing policy of the FoodCollection")

### Including and excluding Range Indexes

Instead of including a range index on every property by default, you can chose to either include or exclude specific paths from the index. Let's go through some simple examples (no need to enter these into the Azure Portal, we can just review them here).

Within the **FoodCollection**, documents have this schema (some properties were removed for simplicity):

```json
{
    "id": "36000",
    "_rid": "LYwNAKzLG9ADAAAAAAAAAA==",
    "_self": "dbs/LYwNAA==/colls/LYwNAKzLG9A=/docs/LYwNAKzLG9ADAAAAAAAAAA==/",
    "_etag": "\"0b008d85-0000-0700-0000-5d1a47e60000\"",
    "description": "APPLEBEE'S, 9 oz house sirloin steak",
    "tags": [
        {
            "name": "applebee's"
        },
        {
            "name": "9 oz house sirloin steak"
        }
    ],

    "manufacturerName": "Applebee's",
    "foodGroup": "Restaurant Foods",
    "nutrients": [
        {
            "id": "301",
            "description": "Calcium, Ca",
            "nutritionValue": 16,
            "units": "mg"
        },
        {
            "id": "312",
            "description": "Copper, Cu",
            "nutritionValue": 0.076,
            "units": "mg"
        },
    ]
}
```

If you wanted to only index the manufacturerName, foodGroup, and nutrients array with a range index, you should define the following index policy. In this example, we use the wildcard character `*` to indicate that we would like to index all paths within the nutrients array.

```json
{
        "indexingMode": "consistent",
        "includedPaths": [
            {
                "path": "/manufacturerName/*"
            },
            {
                "path": "/foodGroup/*"
            },
            {
                "path": "/nutrients/[]/*"
            }
        ],
        "excludedPaths": [
            {
                "path": "/*"
            }
        ]
    }
```

However, it's possible we may just want to index the nutritionValue of each array element.

In this next example, the indexing policy would explicitly specify that the nutritionValue path in the nutrition array should be indexed. Since we don't use the wildcard character `*`, no additional paths in the array are indexed.

```json
{
        "indexingMode": "consistent",
        "includedPaths": [
            {
                "path": "/manufacturerName/*"
            },
            {
                "path": "/foodGroup/*"
            },
            {
                "path": "/nutrients/[]/nutritionValue/*"
            }
        ],
        "excludedPaths": [
            {
                "path": "/*"
            }
        ]
    }
```

Finally, it's important to understand the difference between the `*` and `?` characters.

The `*` character indicates that Azure Cosmos DB should index every path beyond that specific node. 

The `?` character indicates that Azure Cosmos DB should index no further paths beyond this node.

In the above example, there are no additional paths under nutritionValue. If we were to modify the document and add a path here, having the wildcard character `*`  in the above example would ensure that the property is indexed without explicitly mentioning the name.

### Understand query requirements

Before modifying indexing policy, it's important to understand how the data is used in the collection. 

If your workload is write-heavy or your documents are large, you should only index necessary paths. This will significantly decrease the amount of RU's required for inserts, updates, and deletes.

Let's imagine that the following queries are the only read operations that are executed on the **FoodCollection** container.

**Query #1**

```sql
SELECT * FROM c WHERE c.manufacturerName = <manufacturerName>
```

**Query #2**

```sql
SELECT * FROM c WHERE c.foodGroup = <foodGroup>
```

These queries only require that a range index be defined on **manufacturerName** and **foodGroup**, respectively. We can modify the indexing policy to index only these properties.

### Edit the indexing policy by including paths

1. In the Azure Portal, navigate back to the **FoodCollection** container
2. Select the **Scale & Settings** link
3. In the **Indexing Policy** section, replace the existing json file with the following:

    ```json
    {
        "indexingMode": "consistent",
        "includedPaths": [
            {
                "path": "/manufacturerName/*"
            },
            {
                "path": "/foodGroup/*"
            }
        ],
        "excludedPaths": [
            {
                "path": "/*"
            }
        ]
    }
    ```

    > This new indexing policy will create a range index on only the manufacturerName and foodGroup properties. It will remove range indexes on all other properties.

4. Select **Save**. Azure Cosmos DB will update the index in the container, using your excess provisioned throughput to make the updates.

    > During the container re-indexing, write performance is unaffected. However, queries may return incomplete results.

5. In the menu, select the **New SQL Query** icon.
6. Paste the following SQL query and select **Execute Query**:

    ```sql
    SELECT * FROM c WHERE c.manufacturerName = "Kellogg, Co."
    ```

7. Navigate to the **Query Stats** tab. You should observe that this query still has a low RU charge, even after removing some properties from the index. Because the **manufacturerName** was the only property used as a filter in the query, it was the only index that was required.

    ![Query metrics are displayed for the previous query](./images/lab03/03-querymetrics_01.JPG "Review the query metrics")

8. Replace the query text with the following and select **Execute Query**:

    ```sql
    SELECT * FROM c WHERE c.description = "Bread, blue corn, somiviki (Hopi)"
    ```

9. Observe that this query has a very high RU charge even though only a single document is returned. This is because no range index is currently defined for the `description` property.

10. Observe the **Query Metrics**:

    ![Query metrics are displayed for the previous query](./images/lab03/03-querymetrics_02.JPG "Review the query metrics")

    > If a query does not use the index, the **Index hit document count** will be 0. We can see above that the query needed to retrieve 5,187 documents and ultimately ended up only returning 1 document.

### Edit the indexing policy by excluding paths

In addition to manually including certain paths to be indexed, you can exclude specific paths. In many cases, this approach can be simpler since it will allow all new properties in your document to be indexed by default. If there is a property that you are certain you will never use in your queries, you should explicitly exclude this path.

We will create an indexing policy to index every path except for the **description** property.

1. Navigate back to the **FoodCollection** in the Azure Portal
2. Select the **Scale & Settings** link
3. In the **Indexing Policy** section, replace the existing json file with the following:

    ```json
    {
        "indexingMode": "consistent",
        "includedPaths": [
            {
                "path": "/*"
            }
        ],
        "excludedPaths": [
            {
                "path": "/description/*"
            }
        ]
    }
    ```

    > This new indexing policy will create a range index on every property **except** for the description.

4. Select **Save**. Azure Cosmos DB will update the index in the container, using your excess provisioned throughput to make the updates.

    > During the container re-indexing, write performance is unaffected. However, queries may return incomplete results.

5. After defining the new indexing policy, navigate to your **FoodCollection** and select the **Add New SQL Query** icon. Paste the following SQL query and select **Execute Query**:

    ```sql
    SELECT * FROM c WHERE c.manufacturerName = "Kellogg, Co."
    ```

6. Navigate to the **Query Stats** tab. You should observe that this query still has a low RU charge since manufacturerName is indexed.

7. Replace the query text with the following and select **Execute Query**:

    ```sql
    SELECT * FROM c WHERE c.description = "Bread, blue corn, somiviki (Hopi)"
    ```

8. Observe that this query has a very high RU charge even though only a single document is returned. This is because the `description` property is explicitly excluded in the indexing policy.

## Adding a Composite Index

For ORDER BY queries that order by multiple properties, a composite index is required. A composite index is defined on multiple properties and must be manually created.

1. In the **Azure Cosmos DB** blade, locate and select the **Data Explorer** link on the left side of the blade.
2. In the **Data Explorer** section, expand the **NutritionDatabase** database node and then expand the **FoodCollection** container node.
3. Select the icon to add a **New SQL Query**.
4. Paste the following SQL query and select **Execute Query**

    ```sql
    SELECT * FROM c ORDER BY c.foodGroup ASC, c.manufacturerName ASC
    ```

    > This query will fail with the following error:

    ```sql
    "The order by query does not have a corresponding composite index that it can be served from."
    ```

    > In order to run a query that has an ORDER BY clause with one property, the default range index is sufficient. Queries with multiple properties in the ORDER BY clause require a composite index.

5. Still within the **FoodCollection** node, select the **Scale & Settings** link. In the **Indexing Policy** section, you will add a composite index.

6. Replace the **Indexing Policy** with the following text:

    ```json
    {
        "indexingMode": "consistent",
        "automatic": true,
        "includedPaths": [
            {
                "path": "/manufacturerName/*"
            },
            {
                "path": "/foodGroup/*"
            }
        ],
        "excludedPaths": [
            {
                "path": "/*"
            },
            {
                "path": "/\"_etag\"/?"
            }
        ],
        "compositeIndexes": [
            [
                {
                    "path": "/foodGroup",
                    "order": "ascending"
                },
                {
                    "path": "/manufacturerName",
                    "order": "ascending"
                }
            ]
        ]
    }
    ```

7. **Save** this new indexing policy. The update should take approximately 10-15 seconds to apply to your container.

    > This indexing policy defines a composite index that allows for the following ORDER BY queries. Test each of these by running them in your existing open query tab in the **Data Explorer**. When you define the order for properties in a composite index, they must either exactly match the order in the ORDER BY clause or be, in all cases, the opposite value.

8. Run the following queries.

    ```sql
    SELECT * FROM c ORDER BY c.foodGroup ASC, c.manufacturerName ASC
    ```

9. Run the following query, which the current composite index does not support.

    ```sql
    SELECT * FROM c ORDER BY c.foodGroup DESC, c.manufacturerName ASC
    ```

10. This query will not run without an additional composite index. Modify the indexing policy to include an additional composite index.

    ```json
    {
        "indexingMode": "consistent",
        "automatic": true,
        "includedPaths": [
            {
                "path": "/manufacturerName/*"
            },
            {
                "path": "/foodGroup/*"
            }
        ],
        "excludedPaths": [
            {
                "path": "/*"
            },
            {
                "path": "/\"_etag\"/?"
            }
        ],
        "compositeIndexes": [
            [
                {
                    "path": "/foodGroup",
                    "order": "ascending"
                },
                {
                    "path": "/manufacturerName",
                    "order": "ascending"
                }
            ],
            [
                {
                    "path": "/foodGroup",
                    "order": "descending"
                },
                {
                    "path": "/manufacturerName",
                    "order": "ascending"
                }
            ]
        ]
    }
    ```

11. Re-run the query, it should succeed.

> [Learn more about defining composite indexes](https://docs.microsoft.com/en-us/azure/cosmos-db/how-to-manage-indexing-policy#composite-indexing-policy-examples).