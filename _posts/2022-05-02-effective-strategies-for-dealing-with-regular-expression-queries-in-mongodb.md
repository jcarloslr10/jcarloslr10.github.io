---
title: 'Effective strategies for dealing with regular expression queries in MongoDB'
date: 2022-05-02 00:00:00 +0200
categories: [Data, MongoDB]
tags: [database, mongodb, regex, performance, tuning, index]
---

## The Millenium Problem
---
Some of us have worked building applications where, at some point, they need to filter data based on a specified pattern. At first glance, it may seem like a simple problem to solve, but it can hide a poison dart behind it.

It all starts when a stakeholder tells us that the end-user has a need to filter data based on text matching. We, as conscientious software engineer, ask the following questions:

> "Does it have to be an exact text matching?"

> "Does it have to be a **smarter** text matching by taking singular and plural terms into account and skipping stop words?"

If the answer to the **second question is yes**, we should consider paths related to [**text indexes**](https://www.mongodb.com/docs/manual/core/index-text/) in MongoDB or, for more demanding scenarios, other technologies such as [**Atlas Search**](https://www.mongodb.com/docs/atlas/atlas-search/) or [**Elasticsearch**](https://www.elastic.co/what-is/elasticsearch).

If the answer to the **first question is yes** then we should consider use [**regular indexes**](https://www.mongodb.com/docs/manual/indexes/), although we still have one more question to ask:

> "Can the pattern match only against the values from the beginning of the string (**prefix pattern**) or anywhere (**LIKE-style**)?"

Fortunately, if the answer to the question is *"It only matches against the values from the beginning..."* then we just dodged a bullet 😁 because filtering prefix patterns over text index in MongoDB is the least bad thing that can happen to us since it is efficient enough. But, if the answer to the question is *"Anywhere..."*, then we are in trouble. 😥

By the way, before continuing, the stakeholders also ask us to make the matchings insensitive to uppercase, lowercase and diacritics... 😭

## Regex operator

First, I would like to introduce the `$regex` operator in MongoDB. The operator accepts a regular expression of the following form `/pattern/<options>`. An example of non-prefix pattern might be `/cor/i` which looks for the term **cor** anywhere in the string and insensitively.

This type of approach poses two problems, even using regular indexes:

1. The use of non-prefix pattern does not allow setting boundaries when going through the data to be filtered.
2. The option flag `i` causes MongoDB to perform insensitive comparisons which is more inefficient.

## Use case

The application that we are building has a list of users. A end-user of this application has to be able to search users by full name, but to perform meaningful searches on the data, at least one term of **at least three alphanumeric characters** must be typed into the text search box.

This limitation on the minimum size of the term is set because searches for **below three characters** will not yield consistent and useful results.

Examples of **correct** searches:
`Bry`
`Bryan`
`cor Be`

Examples of **incorrect** searches:
`b`
`Br`
`r b`
`or Be`

#### User model

The below JSON represents a user in the `users` collection in MongoDB.

```json
{
  "_id": { "$oid":"626d70bcc30933f61a0786e3"},
  "name":"Corey Batz",
  "job":"Legacy Usability Consultant"
}
```

#### User query

The below Javascript represents how the data are queried using an insensitive non-prefix pattern.

```javascript
db.getCollection("users").aggregate([
    { $match: { name: { $regex: /corey ba/i } } }
])
```

Next, we are going to run the queries on the `users` collection which has **1,000,000 documents**, so stay tuned.

## Avoiding insensitive flag

Building on the user data model above, a new field called `normalizedName` could be added to store the normalized name (lowercase, no diacritics, single space) on which to match later. That may be possible by normalizing the search terms before trying to query them.

So we would have the below JSON representation in the `users` collection:

```json
{
  "_id": { "$oid":"626d70bcc30933f61a0786e3"},
  "name":"Corey Batz",
  "normalizedName":"corey batz",
  "job":"Legacy Usability Consultant"
}
```

And the query would look like this:

```javascript
db.getCollection("users").aggregate([
    { $match: { normalizedName: { $regex: /corey ba/ } } }
])
```

At least this solution will prevent MongoDB from having to perform insensitive comparisons, although it is not enough to guarantee proper operation in a production environment.

It seems we will have to create an index to make the `$regex` expression more efficient because so far MongoDB is scanning the collection. Let's go there!

```javascript
db.getCollection("users").createIndex(
  { normalizedName: 1 },
  { name: "ix_normalizedName" }
);
```

If we run the query again and take a look at the query plan we can see the following:

```json
{
  "explainVersion": "1",
  "queryPlanner": {
    "namespace": "crm.users",
    "indexFilterSet": false,
    "parsedQuery": {
      "normalizedName": {
        "$regex": "rey li"
      }
    },
    "optimizedPipeline": true,
    "maxIndexedOrSolutionsReached": false,
    "maxIndexedAndSolutionsReached": false,
    "maxScansToExplodeReached": false,
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "filter": {
          "normalizedName": {
            "$regex": "rey li"
          }
        },
        "keyPattern": {
          "normalizedName": 1.0
        },
        "indexName": "ix_normalizedName",
        "isMultiKey": false,
        "multiKeyPaths": {
          "normalizedName": []
        },
        "isUnique": false,
        "isSparse": false,
        "isPartial": false,
        "indexVersion": 2.0,
        "direction": "forward",
        "indexBounds": {
          "normalizedName": [
            "[\"\", {})",
            "[/rey li/, /rey li/]"
          ]
        }
      }
    },
    "rejectedPlans": []
  },
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 40.0,
    "executionTimeMillis": 1048.0,
    "totalKeysExamined": 1000000.0,
    "totalDocsExamined": 40.0,
    "executionStages": {
      "stage": "FETCH",
      "nReturned": 40.0,
      "executionTimeMillisEstimate": 93.0,
      "works": 1000001.0,
      "advanced": 40.0,
      "needTime": 999960.0,
      "needYield": 0.0,
      "saveState": 1000.0,
      "restoreState": 1000.0,
      "isEOF": 1.0,
      "docsExamined": 40.0,
      "alreadyHasObj": 0.0,
      "inputStage": {
        "stage": "IXSCAN",
        "filter": {
          "normalizedName": {
            "$regex": "rey li"
          }
        },
        "nReturned": 40.0,
        "executionTimeMillisEstimate": 92.0,
        "works": 1000001.0,
        "advanced": 40.0,
        "needTime": 999960.0,
        "needYield": 0.0,
        "saveState": 1000.0,
        "restoreState": 1000.0,
        "isEOF": 1.0,
        "keyPattern": {
          "normalizedName": 1.0
        },
        "indexName": "ix_normalizedName",
        "isMultiKey": false,
        "multiKeyPaths": {
          "normalizedName": []
        },
        "isUnique": false,
        "isSparse": false,
        "isPartial": false,
        "indexVersion": 2.0,
        "direction": "forward",
        "indexBounds": {
          "normalizedName": [
            "[\"\", {})",
            "[/rey li/, /rey li/]"
          ]
        },
        "keysExamined": 1000000.0,
        "seeks": 1.0,
        "dupsTested": 0.0,
        "dupsDropped": 0.0
      }
    },
    "allPlansExecution": []
  },
  "command": {
    "aggregate": "users",
    "pipeline": [
      {
        "$match": {
          "normalizedName": {
            "$regex": /rey li/
          }
        }
      }
    ],
    "cursor": {},
    "$db": "crm"
  },
  "serverInfo": {
    ...
  },
  "serverParameters": {
    ...
  },
  "ok": 1.0
}
```

Analyzing the query plan we see that MongoDB takes the previously created index as the first stage of the winning plan:

`queryPlanner.winningPlan.inputStage.stage: "IXSCAN"`
`queryPlanner.winningPlan.inputStage.indexName: "ix_normalizedName"`

That's good! But what about the amount of work using the index?

Well, if we see the total number of index key examined, we will be stunned. MongoDB scanned the whole index (**1,000,000 keys**) to end up returning **40 documents** and it took 1,048 milliseconds to execute it. 🥴

`executionStats.executionTimeMillis: 1048.0`

`executionStats.totalKeysExamined: 1000000.0`

`executionStats.totalDocsExamined: 40.0`

`executionStats.nReturned: 40.0`

As the `$regex` is non-prefix expression, MongoDB cannot optimize the `IXSCAN` stage setting bounds when scanning the index, so we need a strategy for MongoDB to be able to set these bounds to do the index scan and return results faster.

## Querying efficient non-prefix pattern

As stated before, it's assumed that at least three alphanumeric character term is typed into text search box in order to perform meaningful searches, so let's define the strategy that will change our lives. 🤨

### The Algorithm 🚀

Since we only perform searches if the search term has three alphanumeric characters, we could add a new field called `normalizedNameChunks` to store all the three-character chunks of the `normalizedName` field including words with less than three characters as well. Let me explain:

Taking the `normalizedName` **"corey li batz"** as an example, we would do the following:

1. Split the `normalizedName` by spaces, outputting: `["corey", "li", "batz"]`
2. Divide each word into as many three-character chunks as possible and keep words less than three characters as is, outputting: `["cor", "ore", "rey", "li", "bat", "atz"]`

```json
{
  "_id": { "$oid":"626d70bcc30933f61a0786e3"},
  "name":"Corey Li Batz",
  "normalizedName":"corey li batz",
  "normalizedNameChunks":["cor", "ore", "rey", "li", "bat", "atz"],
  "job":"Legacy Usability Consultant"
}
```

Using this approach, we could search for matches on this new field, but first we would need to apply the same steps to the search text input.

Taking the search text input **"rey li"** as an example, we would do the following:

1. Split the text input by spaces, outputting: `["rey", "li"]`
2. Divide each word into as many three-character chunks as possible and keep words less than three characters as is, outputting: `["rey", "li"]`

Thus, we could build a query that finds an intersection between the array of the `normalizedNameChunks` field and the array of the text input.

`["cor", "ore", "rey", "li", "bat", "atz"] ∩ ["rey", "li"] = ["rey"]`

Using the `$in` array operator we manage to perform the intersection operation between array field and input array, creating the query as follows:

```javascript
db.getCollection("users").aggregate([
    { $match: { normalizedNameChunks: { $in: ["rey", "li"] } } },
])
```

The intersection operations means that there are matches between the arrays, but the problem with this new query is that it does not guarantee that the order of the chunks is correct, so we need to do something else on the query:

```javascript
db.getCollection("users").aggregate([
    { $match: { normalizedNameChunks: { $in: ["rey", "li"] } } },
    { $match: { normalizedName: { $regex: /rey li/ } } }
])
```

The new stage filtering by `normalizedName` field using `$regex` expression will ensure that the order of the chunks is correct.

One more time as we have done before, we need to create a new index to support the new query conditions on `normalizedNameChunks` and `normalizedName` fields:

```javascript
db.getCollection("users").createIndex(
  { normalizedNameChunks: 1, normalizedName: 1 },
  { name: "ix_normalizedNameChunks_normalizedName" }
);
```

Now we really have it all! Let's see the query plan again.

```json
{
  "explainVersion": "1",
  "queryPlanner": {
    "namespace": "crm.users",
    "indexFilterSet": false,
    "parsedQuery": {
      "$and": [
        {
          "normalizedName": {
            "$regex": "rey li"
          }
        },
        {
          "normalizedNameChunks": {
            "$in": [
              "li",
              "rey"
            ]
          }
        }
      ]
    },
    "optimizedPipeline": true,
    "maxIndexedOrSolutionsReached": false,
    "maxIndexedAndSolutionsReached": false,
    "maxScansToExplodeReached": false,
    "winningPlan": {
      "stage": "FETCH",
      "filter": {
        "normalizedName": {
          "$regex": "rey li"
        }
      },
      "inputStage": {
        "stage": "IXSCAN",
        "keyPattern": {
          "normalizedNameChunks": 1.0,
          "normalizedName": 1.0
        },
        "indexName": "ix_normalizedNameChunks_normalizedName",
        "isMultiKey": true,
        "multiKeyPaths": {
          "normalizedNameChunks": [
            "normalizedNameChunks"
          ],
          "normalizedName": []
        },
        "isUnique": false,
        "isSparse": false,
        "isPartial": false,
        "indexVersion": 2.0,
        "direction": "forward",
        "indexBounds": {
          "normalizedNameChunks": [
            "[\"li\", \"li\"]",
            "[\"rey\", \"rey\"]"
          ],
          "normalizedName": [
            "[\"\", {})",
            "[/rey li/, /rey li/]"
          ]
        }
      }
    },
    "rejectedPlans": []
  },
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 40.0,
    "executionTimeMillis": 278.0,
    "totalKeysExamined": 7018.0,
    "totalDocsExamined": 7016.0,
    "executionStages": {
      "stage": "FETCH",
      "filter": {
        "normalizedName": {
          "$regex": "rey li"
        }
      },
      "nReturned": 40.0,
      "executionTimeMillisEstimate": 207.0,
      "works": 7019.0,
      "advanced": 40.0,
      "needTime": 6977.0,
      "needYield": 0.0,
      "saveState": 17.0,
      "restoreState": 17.0,
      "isEOF": 1.0,
      "docsExamined": 7016.0,
      "alreadyHasObj": 0.0,
      "inputStage": {
        "stage": "IXSCAN",
        "nReturned": 7016.0,
        "executionTimeMillisEstimate": 10.0,
        "works": 7018.0,
        "advanced": 7016.0,
        "needTime": 1.0,
        "needYield": 0.0,
        "saveState": 17.0,
        "restoreState": 17.0,
        "isEOF": 1.0,
        "keyPattern": {
          "normalizedNameChunks": 1.0,
          "normalizedName": 1.0
        },
        "indexName": "ix_normalizedNameChunks_normalizedName",
        "isMultiKey": true,
        "multiKeyPaths": {
          "normalizedNameChunks": [
            "normalizedNameChunks"
          ],
          "normalizedName": []
        },
        "isUnique": false,
        "isSparse": false,
        "isPartial": false,
        "indexVersion": 2.0,
        "direction": "forward",
        "indexBounds": {
          "normalizedNameChunks": [
            "[\"li\", \"li\"]",
            "[\"rey\", \"rey\"]"
          ],
          "normalizedName": [
            "[\"\", {})",
            "[/rey li/, /rey li/]"
          ]
        },
        "keysExamined": 7018.0,
        "seeks": 2.0,
        "dupsTested": 7016.0,
        "dupsDropped": 0.0
      }
    },
    "allPlansExecution": []
  },
  "command": {
    "aggregate": "users",
    "pipeline": [
      {
        "$match": {
          "normalizedNameChunks": {
            "$in": [
              "rey",
              "li"
            ]
          }
        }
      },
      {
        "$match": {
          "normalizedName": {
            "$regex": /rey li/
          }
        }
      }
    ],
    "cursor": {},
    "$db": "crm"
  },
  "serverInfo": {
    ...
  },
  "serverParameters": {
    ...
  },
  "ok": 1.0
}
```

Analyzing the query plan we see that MongoDB takes the previously created index as the first stage of the winning plan:

`queryPlanner.winningPlan.inputStage.stage: "IXSCAN"`
`queryPlanner.winningPlan.inputStage.indexName: "ix_normalizedNameChunks_normalizedName"`

That's awesome! But what about the amount of work using this index?

Well, if we see the total number of index key examined, we will be amazed. MongoDB scanned the index more efficiently setting bounds (**7,018 keys**) to end up returning **40 documents** and it took 278 milliseconds to execute it. Nice! 😎

`executionStats.executionTimeMillis: 278.0`

`executionStats.totalKeysExamined: 7018.0`

`executionStats.totalDocsExamined: 7016.0`

`executionStats.nReturned: 40.0`

## Conclusion

In summary, it is important to keep in mind the following key points:

- Avoid using inefficient regular expression patterns like non-prefix patterns `/batz/` or insensitive patterns `/batz/i`.

- Try to use prefix patterns like `/^corey/` or `/batz$/` when using regular expressions so MongoDB can set bounds for scanning index.

- Try to use a different field to store the normalized value of a given field in order to perform a insensitive comparison in terms of casing and diacritics.

- Splitting words into chunks of a defined minimum length allow us to perform much more efficient first stage.

- An *inefficient* second stage on a much smaller data set is much more *efficient* than an *inefficient* first stage on a very large data set.
