Here’s a guide with sample MongoDB aggregation pipelines to match each pivot scenario we covered earlier using the sales_data collection:


---

Collection: sales_data

Documents look like:

{ "_id": 1, "region": "Asia", "year": 2022, "product": "Laptop", "revenue": 120000 }
...


---

🧩 1. Pivot Only (sum revenue by year)

AG Grid request: pivot on year, no grouping.
Pipeline:

db.sales_data.aggregate([
  { $group: {
      _id: "$year",
      totalRevenue: { $sum: "$revenue" }
    }
  },
  { $sort: { "_id": 1 } },
  { $group: {
      _id: null,
      revenueCols: {
        $push: {
          k: { $concat: [ "revenue_", { $toString: "$_id" } ] },
          v: "$totalRevenue"
        }
      }
    }
  },
  { $project: {
      _id: 0,
      row: { $arrayToObject: "$revenueCols" }
    }
  }
])

This outputs a single document with dynamic fields like revenue_2022, revenue_2023, etc.  


---

🧩 2. Group by region + Pivot by year (sum)

AG Grid request: rowGroupCols: region, pivotCols: year.
Pipeline:

db.sales_data.aggregate([
  { $group: {
      _id: { region: "$region", year: "$year" },
      totalRevenue: { $sum: "$revenue" }
    }
  },
  { $sort: { "_id.region": 1, "_id.year": 1 } },
  { $group: {
      _id: "$_id.region",
      pivot: {
        $push: {
          k: { $concat: [ "revenue_", { $toString: "$_id.year" } ] },
          v: "$totalRevenue"
        }
      }
    }
  },
  { $project: {
      region: "$_id",
      data: { $arrayToObject: "$pivot" },
      _id: 0
    }
  }
])

This outputs one document per region, each with dynamic revenue columns per year.  


---

🧩 3. Group by region + Pivot by year + Filter

AG Grid request: same as above but filter region = "Asia".
Pipeline:

db.sales_data.aggregate([
  { $match: { region: "Asia" } },
  { $group: {
      _id: { region: "$region", year: "$year" },
      totalRevenue: { $sum: "$revenue" }
    }
  },
  { $sort: { "_id.year": 1 } },
  { $group: {
      _id: "$_id.region",
      pivot: {
        $push: {
          k: { $concat: [ "revenue_", { $toString: "$_id.year" } ] },
          v: "$totalRevenue"
        }
      }
    }
  },
  { $project: {
      region: "$_id",
      data: { $arrayToObject: "$pivot" },
      _id: 0
    }
  }
])


---

🧩 4. Multi-Level Grouping: region + product + Pivot by year

AG Grid request: groupKeys empty, rowGroupCols: region & product, pivotCols: year.
Pipeline:

db.sales_data.aggregate([
  { $group: {
      _id: { region: "$region", product: "$product", year: "$year" },
      totalRevenue: { $sum: "$revenue" }
    }
  },
  { $sort: { "_id.region": 1, "_id.product": 1, "_id.year": 1 } },
  { $group: {
      _id: { region: "$_id.region", product: "$_id.product" },
      pivot: {
        $push: {
          k: { $concat: [ "revenue_", { $toString: "$_id.year" } ] },
          v: "$totalRevenue"
        }
      }
    }
  },
  { $project: {
      region: "$_id.region",
      product: "$_id.product",
      data: { $arrayToObject: "$pivot" },
      _id: 0
    }
  }
])


---

🧩 5. Drill-down using groupKeys

When drilling into region = Asia and grouping by product:

db.sales_data.aggregate([
  { $match: { region: groupKeys[0] } }, // e.g. "Asia"
  { $group: {
      _id: { product: "$product", year: "$year" },
      totalRevenue: { $sum: "$revenue" }
    }
  },
  { $sort: { "_id.product": 1, "_id.year": 1 } },
  { $group: {
      _id: "$_id.product",
      pivot: {
        $push: {
          k: { $concat: [ "revenue_", { $toString: "$_id.year" } ] },
          v: "$totalRevenue"
        }
      }
    }
  },
  { $project: {
      product: "$_id",
      data: { $arrayToObject: "$pivot" },
      _id: 0
    }
  }
])


---

🧩 6. Pivot with AVG aggregation

Use $avg instead of $sum:

db.sales_data.aggregate([
  { $group: {
      _id: { region: "$region", year: "$year" },
      avgRevenue: { $avg: "$revenue" }
    }
  },
  { $sort: { "_id.region": 1, "_id.year": 1 } },
  { $group: {
      _id: "$_id.region",
      pivot: {
        $push: {
          k: { $concat: [ "revenue_", { $toString: "$_id.year" } ] },
          v: "$avgRevenue"
        }
      }
    }
  },
  { $project: {
      region: "$_id",
      data: { $arrayToObject: "$pivot" },
      _id: 0
    }
  }
])


---

🧠 Notes & Tips

These pipelines use $group, $push, $arrayToObject, $project stages to dynamically pivot.

Filtering is done with $match.

Sorting before grouping helps stable pivot labels ordering.

$toString is used to generate dynamic key names.

MongoDB does not support SQL-style PIVOT directly, so this is a manual pattern using aggregation.  



---

Need help converting these pipelines into code (Node.js, Spring Data / Java) or making them fully dynamic? I can assist with that too!

Certainly! Here's a consolidated table mapping common AG Grid Pivot Scenarios with their corresponding:

1. AG Grid Server-Side Row Model Request


2. SQL Query


3. MongoDB Aggregation Pipeline




---

✅ Sample sales_data Schema:

Column	Type	Description

region	STRING	Grouping column
year	INT	Pivot column
revenue	DECIMAL	Aggregation (sum)
product	STRING	Optional group column



---

🧩 Scenario 1: Pivot Only on year (no row grouping)

🔸 AG Grid Request:

{
  "pivotMode": true,
  "pivotCols": [{ "id": "year", "field": "year" }],
  "valueCols": [{ "id": "revenue", "field": "revenue", "aggFunc": "sum" }],
  "rowGroupCols": [],
  "groupKeys": []
}

🔹 SQL:

SELECT
  SUM(CASE WHEN year = 2022 THEN revenue ELSE 0 END) AS revenue_2022,
  SUM(CASE WHEN year = 2023 THEN revenue ELSE 0 END) AS revenue_2023,
  SUM(CASE WHEN year = 2024 THEN revenue ELSE 0 END) AS revenue_2024
FROM sales_data;

🔸 MongoDB:

db.sales_data.aggregate([
  { $group: { _id: "$year", total: { $sum: "$revenue" } } },
  { $group: {
      _id: null,
      cols: {
        $push: {
          k: { $concat: ["revenue_", { $toString: "$_id" }] },
          v: "$total"
        }
      }
    }
  },
  { $project: { _id: 0, revenue: { $arrayToObject: "$cols" } } }
])


---

🧩 Scenario 2: Group by region + Pivot by year

🔸 AG Grid Request:

{
  "pivotMode": true,
  "rowGroupCols": [{ "id": "region", "field": "region" }],
  "pivotCols": [{ "id": "year", "field": "year" }],
  "valueCols": [{ "id": "revenue", "field": "revenue", "aggFunc": "sum" }],
  "groupKeys": []
}

🔹 SQL:

SELECT
  region,
  SUM(CASE WHEN year = 2022 THEN revenue ELSE 0 END) AS revenue_2022,
  SUM(CASE WHEN year = 2023 THEN revenue ELSE 0 END) AS revenue_2023,
  SUM(CASE WHEN year = 2024 THEN revenue ELSE 0 END) AS revenue_2024
FROM sales_data
GROUP BY region;

🔸 MongoDB:

db.sales_data.aggregate([
  { $group: {
      _id: { region: "$region", year: "$year" },
      total: { $sum: "$revenue" }
    }
  },
  { $group: {
      _id: "$_id.region",
      pivots: {
        $push: {
          k: { $concat: ["revenue_", { $toString: "$_id.year" }] },
          v: "$total"
        }
      }
    }
  },
  { $project: {
      region: "$_id",
      revenue: { $arrayToObject: "$pivots" },
      _id: 0
    }
  }
])


---

🧩 Scenario 3: Group by region and product + Pivot by year

🔸 AG Grid Request:

{
  "pivotMode": true,
  "rowGroupCols": [
    { "id": "region", "field": "region" },
    { "id": "product", "field": "product" }
  ],
  "pivotCols": [{ "id": "year", "field": "year" }],
  "valueCols": [{ "id": "revenue", "field": "revenue", "aggFunc": "sum" }],
  "groupKeys": []
}

🔹 SQL:

SELECT
  region,
  product,
  SUM(CASE WHEN year = 2022 THEN revenue ELSE 0 END) AS revenue_2022,
  SUM(CASE WHEN year = 2023 THEN revenue ELSE 0 END) AS revenue_2023,
  SUM(CASE WHEN year = 2024 THEN revenue ELSE 0 END) AS revenue_2024
FROM sales_data
GROUP BY region, product;

🔸 MongoDB:

db.sales_data.aggregate([
  { $group: {
      _id: { region: "$region", product: "$product", year: "$year" },
      total: { $sum: "$revenue" }
    }
  },
  { $group: {
      _id: { region: "$_id.region", product: "$_id.product" },
      pivots: {
        $push: {
          k: { $concat: ["revenue_", { $toString: "$_id.year" }] },
          v: "$total"
        }
      }
    }
  },
  { $project: {
      region: "$_id.region",
      product: "$_id.product",
      revenue: { $arrayToObject: "$pivots" },
      _id: 0
    }
  }
])


---

🧩 Scenario 4: Filtering (e.g. Region = 'Asia')

🔸 AG Grid Request:

Same as Scenario 2 + filter model:

{
  "filterModel": {
    "region": { "filterType": "text", "type": "equals", "filter": "Asia" }
  }
}

🔹 SQL:

SELECT
  region,
  SUM(CASE WHEN year = 2022 THEN revenue ELSE 0 END) AS revenue_2022,
  SUM(CASE WHEN year = 2023 THEN revenue ELSE 0 END) AS revenue_2023,
  SUM(CASE WHEN year = 2024 THEN revenue ELSE 0 END) AS revenue_2024
FROM sales_data
WHERE region = 'Asia'
GROUP BY region;

🔸 MongoDB:

db.sales_data.aggregate([
  { $match: { region: "Asia" } },
  { $group: {
      _id: { year: "$year" },
      total: { $sum: "$revenue" }
    }
  },
  { $group: {
      _id: "Asia",
      pivots: {
        $push: {
          k: { $concat: ["revenue_", { $toString: "$_id.year" }] },
          v: "$total"
        }
      }
    }
  },
  { $project: {
      region: "$_id",
      revenue: { $arrayToObject: "$pivots" },
      _id: 0
    }
  }
])


---

🧩 Scenario 5: Drill-Down (Region = 'Asia' → Group by product + Pivot)

🔸 AG Grid Request:

{
  "rowGroupCols": [
    { "id": "region", "field": "region" },
    { "id": "product", "field": "product" }
  ],
  "groupKeys": ["Asia"],
  "pivotCols": [{ "id": "year", "field": "year" }],
  "valueCols": [{ "id": "revenue", "field": "revenue", "aggFunc": "sum" }]
}

🔹 SQL:

SELECT
  product,
  SUM(CASE WHEN year = 2022 THEN revenue ELSE 0 END) AS revenue_2022,
  SUM(CASE WHEN year = 2023 THEN revenue ELSE 0 END) AS revenue_2023,
  SUM(CASE WHEN year = 2024 THEN revenue ELSE 0 END) AS revenue_2024
FROM sales_data
WHERE region = 'Asia'
GROUP BY product;

🔸 MongoDB:

db.sales_data.aggregate([
  { $match: { region: "Asia" } },
  { $group: {
      _id: { product: "$product", year: "$year" },
      total: { $sum: "$revenue" }
    }
  },
  { $group: {
      _id: "$_id.product",
      pivots: {
        $push: {
          k: { $concat: ["revenue_", { $toString: "$_id.year" }] },
          v: "$total"
        }
      }
    }
  },
  { $project: {
      product: "$_id",
      revenue: { $arrayToObject: "$pivots" },
      _id: 0
    }
  }
])


---

Want This as JSON or Excel?

I can format this entire comparison as:

📄 JSON structure for documentation

📊 Excel/CSV with rows per scenario

📦 Ready-to-use code for Spring Boot with MongoDB SQL fallback


Would you like that?

Perfect. Below are consolidated examples for advanced pivot scenarios in AG Grid with matching SQL and MongoDB aggregation pipelines. These go beyond the basics and help build production-grade pivoting support.


---

📊 Scenario 1: Pivot by year and quarter (multi-column pivot)

➤ AG Grid Request

{
  "rowGroupCols": [],
  "pivotCols": [
    { "id": "year", "pivot": true },
    { "id": "quarter", "pivot": true }
  ],
  "valueCols": [{ "id": "revenue", "aggFunc": "sum" }],
  "pivotMode": true,
  "groupKeys": []
}

🧮 SQL Query

SELECT year, quarter, SUM(revenue) AS total_revenue
FROM sales_data
GROUP BY year, quarter

🧩 MongoDB Aggregation

db.sales_data.aggregate([
  {
    $group: {
      _id: { year: "$year", quarter: "$quarter" },
      totalRevenue: { $sum: "$revenue" }
    }
  },
  {
    $group: {
      _id: "$_id.year",
      quarters: {
        $push: {
          k: { $concat: ["revenue_", { $toString: "$_id.quarter" }] },
          v: "$totalRevenue"
        }
      }
    }
  },
  {
    $project: {
      year: "$_id",
      data: { $arrayToObject: "$quarters" },
      _id: 0
    }
  }
])


---

📊 Scenario 2: Pivot with avg, sum, and count together

➤ AG Grid Request

{
  "rowGroupCols": [{ "id": "region" }],
  "pivotCols": [{ "id": "year", "pivot": true }],
  "valueCols": [
    { "id": "revenue", "aggFunc": "sum" },
    { "id": "revenue", "aggFunc": "avg" },
    { "id": "revenue", "aggFunc": "count" }
  ],
  "pivotMode": true,
  "groupKeys": []
}

🧮 SQL Query

SELECT region, year,
       SUM(revenue) AS sum_revenue,
       AVG(revenue) AS avg_revenue,
       COUNT(*) AS count_revenue
FROM sales_data
GROUP BY region, year

🧩 MongoDB Aggregation

db.sales_data.aggregate([
  {
    $group: {
      _id: { region: "$region", year: "$year" },
      sumRevenue: { $sum: "$revenue" },
      avgRevenue: { $avg: "$revenue" },
      countRevenue: { $sum: 1 }
    }
  },
  {
    $group: {
      _id: "$_id.region",
      pivots: {
        $push: {
          k1: { $concat: ["sum_", "$_id.year"] },
          v1: "$sumRevenue",
          k2: { $concat: ["avg_", "$_id.year"] },
          v2: "$avgRevenue",
          k3: { $concat: ["count_", "$_id.year"] },
          v3: "$countRevenue"
        }
      }
    }
  },
  {
    $project: {
      region: "$_id",
      data: {
        $arrayToObject: {
          $concatArrays: [
            { $map: { input: "$pivots", as: "p", in: { k: "$$p.k1", v: "$$p.v1" } } },
            { $map: { input: "$pivots", as: "p", in: { k: "$$p.k2", v: "$$p.v2" } } },
            { $map: { input: "$pivots", as: "p", in: { k: "$$p.k3", v: "$$p.v3" } } }
          ]
        }
      },
      _id: 0
    }
  }
])


---

📊 Scenario 3: Pivot + Pagination (startRow, endRow)

> ⚠️ Note: Pivot + pagination is mostly a front-end concept. The back-end must return all grouped/pivoted data, then AG Grid paginates client-side.



➤ AG Grid Request

{
  "rowGroupCols": [{ "id": "region" }],
  "pivotCols": [{ "id": "year" }],
  "valueCols": [{ "id": "revenue", "aggFunc": "sum" }],
  "pivotMode": true,
  "groupKeys": [],
  "startRow": 0,
  "endRow": 100
}

🧮 SQL Query (with LIMIT/OFFSET)

SELECT region, year, SUM(revenue) AS total
FROM sales_data
GROUP BY region, year
ORDER BY region
LIMIT 100 OFFSET 0

🧩 MongoDB Aggregation (with $skip and $limit)

db.sales_data.aggregate([
  {
    $group: {
      _id: { region: "$region", year: "$year" },
      totalRevenue: { $sum: "$revenue" }
    }
  },
  { $sort: { "_id.region": 1 } },
  { $skip: 0 },
  { $limit: 100 }
])


---

📊 Scenario 4: Pivot with dynamic column values (years)

AG Grid fetches pivot column definitions from metadata.

➤ Sample API to provide pivot columns

GET /api/pivot/values/year → ["2021", "2022", "2023"]

➤ SQL for distinct pivot values

SELECT DISTINCT year FROM sales_data

➤ MongoDB

db.sales_data.distinct("year")


---

📊 Scenario 5: Pivot + Filter (e.g. only Asia, years > 2021)

➤ AG Grid Request

{
  "rowGroupCols": [{ "id": "region" }],
  "pivotCols": [{ "id": "year" }],
  "valueCols": [{ "id": "revenue", "aggFunc": "sum" }],
  "pivotMode": true,
  "groupKeys": [],
  "filterModel": {
    "region": { "type": "equals", "filter": "Asia" },
    "year": { "type": "greaterThan", "filter": 2021 }
  }
}

🧮 SQL Query

SELECT region, year, SUM(revenue) AS total
FROM sales_data
WHERE region = 'Asia' AND year > 2021
GROUP BY region, year

🧩 MongoDB Aggregation

db.sales_data.aggregate([
  {
    $match: {
      region: "Asia",
      year: { $gt: 2021 }
    }
  },
  {
    $group: {
      _id: { region: "$region", year: "$year" },
      totalRevenue: { $sum: "$revenue" }
    }
  }
])


---

Would you like me to package these as a ready-to-use API backend using Spring Boot (with Mongo and/or SQL support), or generate the AG Grid frontend config as well?

