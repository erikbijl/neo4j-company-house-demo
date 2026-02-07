# Neo4j Company House UBO Demo

This repository contains a **demonstration of Ultimate Beneficial Ownership (UBO) and company ownership analysis** using **UK Companies House data**, modeled and explored with **Neo4j Graph Database**.

The demo showcases how **graph technology** makes complex ownership structures, control relationships, and corporate hierarchies easier to understand, query, and visualize compared to traditional relational approaches.

---

## What this demo shows

* Modeling **companies, people, and ownership relationships** as a graph
* Loading the data to **Neo4j Graph** database
* Traversing **multi-level ownership structures**
* Identifying **ultimate beneficial owners**
* Measuring **ownership depth and complexity**
* Visual exploration using **Neo4j Bloom and Dashboards**

This is particularly relevant for:

* Financial crime & AML
* Compliance & KYC
* Corporate structure analysis
* Risk & due diligence

---

## Data Sources
All data comes from **UK Companies House** public datasets. In this register one can find company details, addresses, SIC-codes and ownership relations. You can find the register here: [Search the register
](https://find-and-update.company-information.service.gov.uk/)

For this dataset the following datasets were used:
- **Company data product**: [Free Company Data Product](https://download.companieshouse.gov.uk/en_output.html). Snapshot date used: **2026-01-01**.
- **People with significant control (PSC) data product**: [People with significant control (PSC) snapshot](https://download.companieshouse.gov.uk/en_pscdata.html). Snapshot date used: **2026-01-08**.

For this demo, the focus is specifically on **ownership and control relationships**. Other PSC attributes are intentionally ignored to keep the model clear and focused.

---

## Graph Model Overview

At a high level, the graph consists of:

* **Company, LegalEntity, Individuals, Address and SIC-code** nodes
* **OWNS, REGISTERED_AT and HAS_SIC** relationships

Running the following query shows the model in the database: 


```cypher 
CALL db.schema.visualization()
```
![Neo4j Graph Schema](images/data_model.png)

This model enables:

* Variable-length ownership traversal
* Detection of root / top-level entities
* Identification of ownership chains and loops
* Efficient aggregation across deep hierarchies

---

## Database Backup

The full Neo4j database backup is **too large for GitHub**.

You can download it here: [Google Drive link](https://drive.google.com/file/d/1ftc1a37PjrDVgJTqE-fTzEcJ97vlMxsh/view?usp=drive_link)

---

## Loading Data into Neo4j

The repository contains two Jupyter notebooks used to transform and load the data into Neo4j. This is done using the [Spark Connector for Neo4j](https://neo4j.com/docs/spark/current/). In the notebooks the following is done: 

### 1. Company Data

**`company_house_companies.ipynb`**

* Company details
* Registered Addresses
* SIC codes
* Basic normalization and transformation

### 2. PSC Data

**`company_house_psc.ipynb`**

* People with Significant Control
* Ownership relationships (`OWNS`)
* Focuses only on **ownership and control**
* Non-essential PSC attributes are ignored for clarity

---

## Example Cypher Queries

Below are example queries used in the demo to explore ownership structures.

### Subsidiary tree for a single company

```cypher
MATCH p=(e:Company {company_number: "05310128"})-[:OWNS*]->(:Company)
WHERE all(r IN relationships(p) WHERE coalesce(r.ownership_pct_min, 0) >= 0)
RETURN DISTINCT p
```

![Subsidiaries](images/subsidiaries.png)

---

### Identify top-level companies and ownership depth

```cypher
MATCH (e:Company)-[r:OWNS]->()
WHERE NOT EXISTS((:Company)-[:OWNS]->(e))
WITH DISTINCT e
MATCH p=(e)-[:OWNS*]->(:Company)
WITH e, p, MAX(LENGTH(p)) AS max_depth
UNWIND nodes(p) AS n
WITH e,
     COALESCE(SIZE(COLLECT(DISTINCT n)), 0) AS member_count,
     COALESCE(MAX(max_depth), 0) AS max_depth
RETURN e.company_number, e.name, member_count, max_depth
ORDER BY max_depth DESC
LIMIT 200
```

| company_number | name                              | member_count | max_depth |
|---------------:|-----------------------------------|-------------:|----------:|
| 02055886       | HG POOLED MANAGEMENT LIMITED       | 107          | 21        |
| Oc302604       | Tdr Capital Llp                    | 474          | 20        |
| 0100-01-115658 | Park24 Co., Ltd.                   | 71           | 19        |
| 202201590      | Liberty Global Ltd                 | 172          | 18        |
| Oc392495       | Hilton Worldwide Holding Llp       | 44           | 18        |
| 12319244       | LGC SCIENCE CORPORATION LIMITED    | 30           | 18        |
| 6010001115658  | Park24 Co., Ltd.                   | 70           | 18        |
| 10003599       | CKA HOLDINGS UK LIMITED            | 40           | 18        |
| 5254001        | Arqiva Broadcast Holdings Limited  | 60           | 17        |
| A 28015865     | Telefonica S A                    | 134          | 17        |


---

### Reverse ownership exploration

```cypher
MATCH p=(:Company {company_number: "02852924"})-[:OWNS*]-()
WHERE all(r IN relationships(p) WHERE coalesce(r.ownership_pct_min, 0) >= 0)
RETURN p
```

![Circular Ownership Structure](images/circular_ownership.png)

---

### Ownership traversal using APOC

```cypher
MATCH (e:Company {company_number: "00521970"})
CALL apoc.path.expandConfig(e, {
  relationshipFilter: "OWNS",
  minLevel: 1,
  maxLevel: -1,
  uniqueness: "RELATIONSHIP_GLOBAL",
  filterStartNode: false
})
YIELD path
RETURN DISTINCT path
```

![All reachable companies in Corporate Group](images/corporate_group.png)


---

### Most connected legal entities

```cypher
MATCH (x:LegalEntity)-[]-(c:Company)
WITH x.name AS name, COUNT(c) AS count
RETURN name, count
ORDER BY count DESC
LIMIT 10
```

| name                                                     | count |
|----------------------------------------------------------|------:|
| China Investment Corporation                              | 74    |
| His Majesty The King In Right Of Alberta (Hmka)           | 61    |
| Qatar Investment Authority                                | 26    |
| Scottish Enterprise                                      | 24    |
| Government Of Abu Dhabi                                   | 20    |
| Qa Directors Limited                                      | 20    |
| British Columbia Investment Management Corporation        | 19    |
| Minister For Finance Singapore                            | 19    |
| Knotel Uk Ltd.                                            | 16    |
| Ipswich Borough Council                                   | 15    |


---

## Visualizations

### Neo4j Dashboards

Used for aggregated insights such as:

* Ownership depth
* Number of subsidiaries
* Highly connected entities

![Company Dashboard](images/dashboard.png)

---

### Neo4j Bloom

Used for **interactive exploration** of:

* Ownership chains
* Control relationships
* Complex corporate structures

Bloom makes it easy to visually navigate UBO relationships without writing Cypher.

![Bloom](images/bloom.png)

---

## Disclaimer

This repository is intended as a **technical demo** only.

* Data is sourced from public Companies House snapshots
* The model is simplified for demonstration purposes
* This should **not** be used for legal or compliance decisions without additional validation
