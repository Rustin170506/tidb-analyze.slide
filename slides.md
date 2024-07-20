---
theme: default
background: "#white"
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## TiDB Analyze
  Analyze collects statistics for the optimizer to generate better query plans.

  Learn more at [TiDB](https://docs.pingcap.com/tidb/stable/sql-statement-analyze-table)
drawings:
  persist: false
defaults:
  foo: true
transition: slide-left
title: TiDB Analyze
schemaColor: white
mdc: true
---

# TiDB Analyze

A Deep Dive

Based on TiDB [v8.1.0](https://github.com/pingcap/tidb/tree/v8.1.0)

RUSTIN LIU

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Press Space to Start
  </span>
</div>


<div class="abs-br m-6 flex gap-2">
  <button @click="$slidev.nav.openInEditor()" title="Open in Editor" class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon:edit />
  </button>
  <a href="https://github.com/slidevjs/slidev" target="_blank" alt="GitHub" title="Open in GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!---
Thanks for joining me today. I'm Rustin, today I'm gonna talk about TiDB Analyze.
TiDB Analyze is a feature that collects statistics for the optimizer to generate better query plans.
If you want to take a look at the source code, you need to go to the TiDB repository and check out the v8.1.0 tag.
All my examples and algorithms are based on this version.
Alright, let's get started.
-->

---
transition: slide-up
---

# Rustin Liu

<div class="leading-8 opacity-80">
PingCAP Database Engineer.<br/>
Cargo/Crates.io/Rustup Maintainer.<br/>
Tokio Console Maintainer.<br/>
</div>

<div my-10 grid="~ cols-[40px_1fr] gap-y4" items-center justify-center>
  <div i-ri-github-line op50 ma text-xl/>
  <div><a href="https://github.com/hi-rustin" target="_blank">hi-rustin</a></div>
  <div i-ri-firefox-line op50 ma text-xl/>
  <div><a href="https://hi-rustin.rs" target="_blank">hi-rustin.rs</a></div>
</div>

<img src="https://avatars.githubusercontent.com/u/29879298?v=4" rounded-full w-30 abs-tr mt-22 mr-22/>

<div flex="~ gap2">
</div>

<!---
First, let me introduce myself. I'm Rustin Liu, a Database Engineer at PingCAP.
I'm also pretty active in the Rust community. I maintain the Cargo, Crates.io, and Rustup projects.
I'm also a maintainer of the Tokio Console project.
-->

---
transition: slide-up
layout: center
---

<div text-6xl fw100>
  Agenda
</div>

<br>

<div class="grid grid-cols-[3fr_2fr] gap-4">
  <div class="border-l border-gray-400 border-opacity-25 !all:leading-12 !all:list-none my-auto">

  - Analyze Overview
  - Data Structure Overview
  - Data Flow Overview
  - Data Structure & Data Flow (TiKV Perspective)
  - Data Structure & Data Flow (TiDB Perspective)
  - Q&A

  </div>
</div>

<!---

This is the agenda for today's talk.
We will start with an overview of the Analyze feature.
Then we will dive into the data structure and data flow overview.
We will look at the data structure and data flow from both the TiKV and TiDB perspectives.
Finally, we will have a Q&A session.

-->

---
transition: slide-left
layout: center
---

# Analyze Overview

---
transition: slide-left
layout: default
---

# Analyze Statement

<img src="/analyze.svg" />

<!---

Alright, let’s break down the basic syntax of the Analyze statement.

First off, you can analyze different parts of your database: tables, partitions, columns, and indexes.

You also have the option to focus only on predicate columns. This means it will analyze the columns used in the conditions of your queries.

Plus, you can tweak a few settings, like specifying the number of top N items or the number of buckets.

Now, let’s dive into some examples to see how it works.

-->


---
transition: slide-left
---

# Analyze Statement


```sql{all|1-2|4-5|7-8|10-11|13-14|16-17|19-20|22-23}
-- Analyze Tables
ANALYZE TABLE t1, t2;

-- Analyze Partitions
ANALYZE TABLE t PARTITION p1, p2;

-- Analyze Columns
ANALYZE TABLE t COLUMNS c1, c2;

-- Analyze Indexes
ANALYZE TABLE t INDEX idx1, idx2;

-- Analyze Partitions' Columns
ANALYZE TABLE t PARTITION p1 COLUMNS c1, c2;

-- Analyze Partitions' Indexes
ANALYZE TABLE t PARTITION p1 INDEX idx1, idx2;

-- Analyze Predicate Columns
ANALYZE TABLE t PREDICATE COLUMNS;

-- Analyze With Only 20 Top N
ANALYZE TABLE t COLUMNS c1, c2 WITH 20 TOPN;
```

---
transition: slide-up
layout: center
---

# Data Structure Overview

<!---

After executing the Analyze statement, we’ll generate some statistics.

In this section, we’ll examine the data structure used to store these statistics.

-->

---
transition: slide-left
---

# Data Structure Overview
A simple example.

Create table
```sql{all|2}
use test;
create table t (a int);
```
Insert 2000 rows
```ts{all|6-9}
import { Client } from "https://deno.land/x/mysql/mod.ts";

const client = await new Client().connect({...});

for (let i = 0; i < 2000; i++) {
  await client.execute(`INSERT INTO t (a) VALUES (?)`, [i]);
  if (i % 2 === 0) {
    await client.execute(`INSERT INTO t (a) VALUES (?)`, [i]);
  }
}

await client.close();
```

<!---

Let’s start with a simple example.

First, we’ll create a table named t with a single column a.

Next, we’ll insert 2000 rows into the table.

Notice that we insert the same value twice for every even number.

Keep this example in mind, as we’ll use it later to illustrate the data structure.

-->


---
transition: slide-left
---

# Data Structure Overview
Column Selectivity

```sql
explain select * from t where a = 100;
```

| id                 | estRows                                                                              | task        | access object | operator info       |
| :----------------- | :----------------------------------------------------------------------------------- | :---------- | :------------ | :------------------ |
| TableReader\_7     | 2.00                                                                                 | root        |               | data:Selection\_6   |
| └─Selection\_6     | <span class="text-green-500" v-mark="{ color: 'green', type: 'circle' }">2.00</span> | cop\[tikv\] |               | eq\(test.t.a, 100\) |
| └─TableFullScan\_5 | 3000.00                                                                              | cop\[tikv\] | table:t       | keep order:false    |

<br/>

```go{all|2}
func equalRowCountOnColumn(encodedVal []byte...) {
  rowcount, ok := c.TopN.QueryTopN(sctx, encodedVal)
	if ok {
		return float64(rowcount), nil
	}
}
```

<!---

Let’s look at a simple query that selects rows where column A equals 100.

This is a basic example of column selectivity, using a simple equality condition. The optimizer estimates that there are 2 rows that match this condition.

The optimizer uses the TopN data structure to make this estimate.

Since the value 100 is in the TopN data structure, the optimizer can quickly determine the number of matching rows.

I want to emphasize that this is a very straightforward example. For more complex queries, the optimizer’s use of statistics is much more intricate.

-->

---
transition: slide-left
---

# Data Structure Overview
Column Selectivity

TopN

```sql
select * from mysql.stats_top_n order by value limit 5;
```

| table\_id | is\_index | hist\_id | value                                                                                                                                                                             | count |
| :-------- | :-------- | :------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---- |
| 106       | 0         | 1        | <span class="text-green-500" v-mark="{ color: 'green', type: 'circle' }">0x038</span>00000000000<span class="text-red-500" v-mark="{ color: 'red', type: 'circle' }" >0000</span> | 2     |
| 106       | 0         | 1        | 0x038000800000000002                                                                                                                                                              | 2     |
| 106       | 0         | 1        | 0x038000800000000004                                                                                                                                                              | 2     |
| 106       | 0         | 1        | 0x038000800000000006                                                                                                                                                              | 2     |
| 106       | 0         | 1        | 0x038000800000000008                                                                                                                                                              | 2     |


<!---

We use a system table called stats_top_n to store the TopN data structure.

If we query this table, we can see the top N values and their counts.

For example, we can see that the value 0x0380000000000000 is in the TopN data structure with a count of 2.

The 0x038 indicates that the value is an integer.

In this example, the data skew isn’t very high, so all values have the same count.

-->


---
transition: slide-left
---

# Data Structure Overview
Column Selectivity

```sql
explain select * from t where a = 1999;
```

| id                 | estRows                                                                          | task        | access object | operator info        |
| :----------------- | :------------------------------------------------------------------------------- | :---------- | :------------ | :------------------- |
| TableReader\_7     | 1.00                                                                             | root        |               | data:Selection\_6    |
| └─Selection\_6     | <span class="text-red-500" v-mark="{ color: 'red', type: 'circle' }">1.00</span> | cop\[tikv\] |               | eq\(test.t.a, 1999\) |
| └─TableFullScan\_5 | 3000.00                                                                          | cop\[tikv\] | table:t       | keep order:false     |

<br/>

```go{all|2,3,4}
func equalRowCountOnColumn(encodedVal []byte...) {
	histCnt, matched := c.Histogram.EqualRowCount(sctx, val, true)
	if matched {
		return histCnt, nil
	}
}
```

<!---

Now, let’s look at another example.

This time, we’re selecting rows where column a equals 1999.

Since we only collect the first top 500/100 values, the value 1999 isn’t in the TopN data structure.

In this case, the optimizer uses the Histogram data structure to estimate the number of matching rows.

The Histogram data structure shows the distribution of values in the column. By analyzing the histogram, the optimizer can estimate how many rows meet the condition.

So, in this example, the optimizer estimates that only 1 row matches the condition.

-->

---
transition: slide-left
---

# Data Structure Overview
Column Selectivity

```sql
select hist_id, bucket_id, count, repeats,
       CAST(lower_bound AS SIGNED) AS lower_bound,
       CAST(upper_bound AS SIGNED) AS upper_bound,
       ndv
from mysql.stats_buckets order by lower_bound desc limit 5;
```

| hist\_id | bucket\_id | count                                                                         | repeats                                                                       | lower\_bound                                                                     | upper\_bound                                                                     | ndv  |
| :------- | :--------- | :---------------------------------------------------------------------------- | :---------------------------------------------------------------------------- | :------------------------------------------------------------------------------- | :------------------------------------------------------------------------------- | :--- |
| 1        | 229        | 1                                                                             | 1                                                                             | 1999                                                                             | 1999                                                                             | 0    |
| 1        | 228        | <span class="text-red-500" v-mark="{ color: 'red', type: 'circle' }">9</span> | <span class="text-red-500" v-mark="{ color: 'red', type: 'circle' }">2</span> | <span class="text-red-500" v-mark="{ color: 'red', type: 'circle' }">1993</span> | <span class="text-red-500" v-mark="{ color: 'red', type: 'circle' }">1998</span> | 0    |
| 1        | 227        | 9                                                                             | 2                                                                             | 1987                                                                             | 1992                                                                             | 0    |
| 1        | 226        | 9                                                                             | 2                                                                             | 1981                                                                             | 1986                                                                             | 0    |
| 1        | 225        | 9                                                                             | 2                                                                             | 1975                                                                             | 1980                                                                             | 0    |


<!---

We use a system table called stats_buckets to store the Histogram data structure.

If we query this table, we can see the histogram buckets.

In this example, we can see that the value 1999 is in a histogram bucket with a count of 1.

The lower and upper bounds of this bucket are both 1999.

If we look at the previous bucket, we see it has a count of 9, with a lower bound of 1993.

Additionally, the repeats are 2, meaning there are two repeated values of the upper bound.

We can use this repeated value to estimate the selectivity of the column for an equality condition.

-->


---
transition: slide-left
---
# Data Structure Overview
Histogram Bucket

- Bucket ID: The bucket ID of the histogram.
- Count: The number of values till the bucket.(**cumulative**)
- Repeats: The number of repeated values at the upper bound.
- Lower Bound: The lower bound of the bucket.
- Upper Bound: The upper bound of the bucket.
- NDV: The number of distinct values in the bucket.

````md magic-move
```json
{
    "bucket_id": 228,
    "count": 9,
    "repeats": 2,
    "lower_bound": 1993,
    "upper_bound": 1998,
    "ndv": 0
}
```


```json
{
    "bucket_id": 228,
    "count": [1993, 1994, 1994, 1995, 1996, 1996, 1997, 1998, 1998],
    "repeats": [1998, 1998],
}
```
````

<!---

Let’s take a closer look at the histogram bucket.

Bucket ID: The ID of the histogram bucket.

Count: The number of values in the bucket. This is a cumulative count, including all values up to the bucket.

Repeats: The number of repeated values at the upper bound.

Lower Bound: The lower bound of the bucket.

Upper Bound: The upper bound of the bucket.

NDV (Number of Distinct Values): The number of distinct values in the bucket.

In this example, we have a bucket with an ID of 228. The count is 9, and the repeats are 2. The lower bound is 1993, and the upper bound is 1998.

If we break down the count and repeats, we can see the individual values within the bucket.

-->

---
transition: slide-left
---


# Data Structure [^1]


<Histogram className="histogram"/>

[^1]: [Piatetsky-Shapiro, Gregory, and Charles Connell. "Accurate Estimation Of The Number Of Tuples Satisfying A Condition"](https://dl.acm.org/doi/pdf/10.1145/971697.602294)

<style>
.histogram {
  height: 80%;
}
.footnotes-sep {
  margin-top: 1em;
}
.footnote-item{
  font-size: 0.5em;
}
.footnote-backref {
  display: none
}
</style>

<!---

As you can see, we use a equi-height histogram to store the data.

The reason we use an equi-height histogram is that it provides a more accurate estimation of the selectivity of the column.

You can find more at this paper.

-->

---
transition: slide-left
---

# Data Structure Overview
Column Selectivity

```sql
explain select * from t where a = 9999;
```

| id                 | estRows                                                                          | task        | access object | operator info        |
| :----------------- | :------------------------------------------------------------------------------- | :---------- | :------------ | :------------------- |
| TableReader\_7     | 1.33                                                                             | root        |               | data:Selection\_6    |
| └─Selection\_6     | <span class="text-red-500" v-mark="{ color: 'red', type: 'circle' }">1.33</span> | cop\[tikv\] |               | eq\(test.t.a, 2000\) |
| └─TableFullScan\_5 | 3000.00                                                                          | cop\[tikv\] | table:t       | keep order:false     |

<br/>

```go{all|2,6}
func equalRowCountOnColumn(encodedVal []byte...) {
	histNDV := float64(c.Histogram.NDV - int64(c.TopN.Num()))
	if histNDV <= 0 {
		return 0, nil
	}
	return c.Histogram.NotNullCount() / histNDV, nil
}
```

<!---

Now, let’s look at another example.

This time, we’re selecting rows where column a equals 9999. Since the value 9999 isn’t in the TopN data structure or the Histogram data structure, the optimizer uses a different method to estimate the number of matching rows.

In this case, the optimizer estimates that 1.33 rows match the condition.

The optimizer uses the NotNullCount and NDV from the Histogram data structure to make this estimate.

-->

---
transition: slide-left
---

# Data Structure Overview
Column Selectivity

- Not Null Count: The number of not null values in the column.
- NDV: The number of distinct values in the column.

<br/>

### How to calculate the NDV(Non-Distinct Value)?

We use [FMSketch(Flajolet-Martin Sketch)](https://en.wikipedia.org/wiki/Flajolet%E2%80%93Martin_algorithm) to calculate the NDV.


<!---

The NotNullCount is the number of not null values in the column. The NDV (Number of Distinct Values) is the number of distinct values in the column.

To calculate the NDV, we use the Flajolet-Martin Sketch algorithm. This algorithm provides an efficient way to estimate the number of distinct values in a set.

Before we dive into the details of the algorithm, let’s take a look at the data flow from the TiKV perspective.

-->

---
transition: slide-up
layout: center
---

# Data Flow Overview

---
transition: slide-left
---

# Data Flow Overview


```plantuml
@startuml

"TiDB Owner/Client" as TC -> TiDB: execute analyze statement
TiDB -> TiKV1: send analyze gRPC request
TiKV1 -> TiKV1: collect statistics
TiKV1 --> TiDB: send analyze gRPC response
TiDB -> TiKV2: send analyze gRPC request
TiKV2 -> TiKV2: collect statistics
TiKV2--> TiDB: send analyze gRPC response
TiDB -> TiDB: build and merge statistics \nand update statistics to system tables
TiDB --> TC: return success
@enduml
```

---
transition: slide-up
layout: center
---

# Data Structure & Data Flow
TiKV Perspective

In TiKV, we only do two things:

1. Calculate the FMSketch.
2. Sample the data.


---
transition: slide-left
---

# Data Structure - FMSketch
TiKV Perspective

Mathematical Assumptions [^1]
1. **Independence of Hash Functions**:
   - Assume a good hash function **h(x)** that uniformly distributes input elements over a large range of integers.
2. **Expectation of Trailing Zeros in Hash Values**:
   - For uniformly distributed hash values, the number of trailing zeros in their binary representation follows a geometric distribution.

[^1]: [Flajolet, Philippe; Martin, G. Nigel (1985). "Probabilistic counting algorithms for data base applications"](https://algo.inria.fr/flajolet/Publications/FlMa85.pdf)

<style>
.footnotes-sep {
  margin-top: 1em;
}
.footnote-item{
  font-size: 0.5em;
}
.footnote-backref {
  display: none
}
</style>

---
transition: slide-left
---

# Data Structure - FMSketch
TiKV Perspective

Algorithm Principles
1. **Hash Mapping**:
   - Map each element of the set to an integer using the hash function h(x).
2. **Trailing Zeros Counting**:
   - For each hash value, count the number of trailing zeros in its binary representation. Record the maximum count **R**.
3. **Cardinality Estimation**:
   - Use the maximum trailing zero count **R** to estimate the cardinality of the set with the formula $2^R$.

---
transition: slide-left
---

# Data Structure - FMSketch
TiKV Perspective

Example
- Given a set $\{a, b, c\}$
- Hash values $h(a) = 8$, $h(b) = 12$, $h(c) = 5$
- Binary representations: $1000$, $1100$, $0101$
- Trailing zeros: $3$, $2$, $0$
- Maximum trailing zeros $R = 3$
- Estimated cardinality: $2^3 = 8$


---
transition: slide-left
---

<FMSketch/>

---
transition: slide-left
---

# Data Structure - FMSketch

<BadFMSketch/>


---
transition: slide-left
---

# Data Structure - Distinct Sampling
TiKV Perspective

Mathematical Assumptions [^1]
1. **Decreasing Probabilities:**
   - Use decreasing probabilities derived from FM sketches.
2. **Hash Function Distribution:**
   - Hash function h with $Pr[h(i) = j] = 2^{-j}$ , ensuring uniform distribution over levels.

[^1]: [Phillip B. Gibbons. "Distinct Sampling for Highly-Accurate Answers
to Distinct Values Queries and Event Reports"](https://www.vldb.org/conf/2001/P541.pdf)

<style>
.footnotes-sep {
  margin-top: 1em;
}
.footnote-item{
  font-size: 0.5em;
}
.footnote-backref {
  display: none
}
</style>

---
transition: slide-left
---

# Data Structure - Distinct Sampling
TiKV Perspective

Algorithm Principles
1. **Maintaining the Sample:**
   - Keep a set of at most k items from the input.
   - Maintain an integer variable l to record the current sampling level.
2. **Hash Function Usage:**
   - Each input item is hashed using function $h$.
   - Include items in the sample if $h(i) \geq l$.

---
transition: slide-left
---

# Data Structure - Distinct Sampling
TiKV Perspective

Algorithm Steps

1. **Initial Sampling:**
   - Start with $l = 1$; all distinct items are sampled.
2. **Sample Pruning:**
   - When sample exceeds k items, increase $l$ by 1.
   - Prune items with hash values less than the current level $l$.
3. **Adjusting Sampling Rate:**
	•	Each increase in *l* halves the sampling rate.
	•	Sample size expected to decrease to approximately $k/2$.

---
transition: slide-left
---

# Data Structure - Distinct Sampling
TiKV Perspective

**Example**: Given k = 3

**Levels:**
1. Level 1: $\{3, 6, 7, 8, 10, 14, 18, 19, 20\}$
2. Level 2: $\{3, 8, 10, 14, 20\}$
3. Level 3: $\{3, 10, 14\}$
4. Level 4: $\{14\}$

**Process:**
- Start at Level 1: All items are sampled.
- Move to Level 2: Items 6, 7, 18, 19 are pruned.
- Move to Level 3: Items 8, 20 are pruned.
-	Move to Level 4: Items 3, 10 are pruned; only item 14 remains.

---
transition: slide-left
---

# Data Structure - Distinct Sampling
TiKV Perspective

Estimation and Analysis

1. **Cardinality Estimation:**
   - Estimate distinct items as $s \cdot 2^l$, where s is current sample size.
2. **Variance Behavior:**
   - With $k$ items, variance grows with $1/\sqrt{k}$.
3. **Accuracy:**
   - Setting $k = O(1/\epsilon^2)$ achieves relative error $\epsilon$ with constant probability.
   - Using parallel repetitions and taking the median estimate reduces failure probability.


---
transition: slide-left
---

# Data Structure - Bernoulli Sampling
General Perspective

Mathematical Assumptions
1. **Independence of Sample Selection**:
   - Each sample in the data set is selected independently from other samples.
2. **Uniform Sampling Probability**:
   - Each sample is selected with a fixed probability $p(0 ≤ p ≤ 1)$, uniformly across the entire data set.
3. **Bernoulli Distribution**:
   - Each sample selection follows a Bernoulli distribution with parameter $p$.

---
transition: slide-left
---

# Data Structure - Bernoulli Sampling
General Perspective

Algorithm Principles
1. **Probability Definition**:
   - Define a sampling probability $p$ for selecting each sample.
2. **Independent Sampling**:
   - For each sample in the data set, generate a random number and compare it to $p$. If the random number is less than $p$, include the sample in the resulting subset.
3. **Subset Generation**:
   - The subset of samples selected using this method forms the result of the Bernoulli sampling.


---
transition: slide-left
---

<BernoulliSampling/>

---
transition: slide-up
layout: center
---

# Data Structure & Data Flow
TiDB Perspective

In TiDB, we do the following things:

1. Merge all FMSketches and Sample Data.
2. Build TopN and Histogram.
3. Update statistics to system tables.



---
transition: slide-left
---

# Data Structure & Data Flow
Overview

```plantuml{ scale: 0.9 }
@startuml



"TiDB Owner/Client" as TC -> TiDB: execute analyze statement
TiDB -> TiKV1: send analyze gRPC request
TiKV1 -> TiKV1: collect statistics
TiKV1 --> TiDB: send analyze gRPC response
TiDB -> TiKV2: send analyze gRPC request
TiKV2 -> TiKV2: collect statistics
TiKV2--> TiDB: send analyze gRPC response
TiDB -> TiDB: build and merge statistics \nand update statistics to system tables
TiDB --> TC: return success
@enduml
```

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Analyze tables or partitions concurrently

```plantuml
@startuml
"Analyze Plan Builder" as APB -> APB: build a analyze plan and create tasks \nfor each table/partition
APB -> "Analyze Executor" as AE: store the analyze plan
AE --> APB: success

group concurrently analyze tables/partitions
AE -> analyzeWorker1: start the analyze worker1 and \nwait for the analyze task
analyzeWorker1 -> analyzeWorker1: analyze the table/partition \nand send the analyze result to the result channel
AE -> analyzeWorker2: start the worker2 and \nwait for the analyze task
analyzeWorker2 -> analyzeWorker2: analyze the table/partition \nand send the analyze result to the result channel
AE -> resultHandler: start the result handler and \nwait for the result channel
resultHandler -> resultHandler: receive the analyze result \nand update statistics to system tables
AE -> AE: wait for all workers to finish
end
AE -> AE: merge global statistics if needed
AE -> AE: update statistics cache
@enduml
```

<div v-click class="absolute top-40 right-0 transform rotate-30 bg-red-500 text-white font-bold py-1 px-1 rounded-lg"> tidb_build_stats_concurrency </div>

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Scan regions concurrently

```plantuml
@startuml
"Analyze Executor" as AE -> AE: open
group concurrently scan regions
  AE -> coprWorker1: start the worker1 and wait for the scan task
  AE -> coprWorker2: start the worker2 and wait for the scan task
  coprWorker1 -> TiKV1: send analyze gRPC request to scan region1
  coprWorker2 -> TiKV1: send analyze gRPC request to scan region2
  TiKV1 -> TiKV1: scan region1 and collect statistics
  TiKV1 -> TiKV1: scan region2 and collect statistics
  TiKV1 --> coprWorker1: send analyze gRPC region1 response
  TiKV1 --> coprWorker2: send analyze gRPC region2 response
end
AE -> AE: build and merge statistics \nand return the analyze result
@enduml
```

<div v-click class="absolute top-40 right-0 transform rotate-30 bg-red-500 text-white font-bold py-1 px-1 rounded-lg"> tidb_analyze_distsql_scan_concurrency </div>

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Merge FMSketches and Sample Data

```plantuml { scale: 0.9 }
@startuml


"Analyze Executor" as AE -> AE: create a root row sample collector
group concurrently merge FMSketches and Sample Data
  AE -> worker1: start the worker1 and \nwait for the row sample collector data channel
  worker1 -> worker1: merge the row sample collector \ndata to the result channel
  AE -> worker2: start the worker2 and \nwait for the row sample collector data channel
  worker2 -> worker2: merge the row sample collector \ndata to the result channel
end
AE -> AE: merge the result channel data to the root row sample collector
AE -> AE: build the TopN and Histogram \nand return analyze result
@enduml
```

<div v-click class="absolute top-40 right-0 transform rotate-30 bg-red-500 text-white font-bold py-1 px-1 rounded-lg"> tidb_build_sampling_stats_concurrency </div>

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Merge Collector

```go{all|3-5|12}
func (s *BernoulliRowSampleCollector) MergeCollector(subCollector RowSampleCollector) {
	s.Count += subCollector.Base().Count
	for i := range subCollector.Base().FMSketches {
		s.FMSketches[i].MergeFMSketch(subCollector.Base().FMSketches[i])
	}
	for i := range subCollector.Base().NullCount {
		s.NullCount[i] += subCollector.Base().NullCount[i]
	}
	for i := range subCollector.Base().TotalSizes {
		s.TotalSizes[i] += subCollector.Base().TotalSizes[i]
	}
	s.baseCollector.Samples = append(s.baseCollector.Samples, subCollector.Base().Samples...)
	s.MemSize += subCollector.Base().MemSize
}
```

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Merge FMSketches

```go{all|3-13|14-18}
func (s *FMSketch) MergeFMSketch(rs *FMSketch) {
	...
  // Pick the bigger mask.
	if s.mask < rs.mask {
		s.mask = rs.mask
		s.hashset.Iter(func(key uint64, _ bool) bool {
			if (key & s.mask) != 0 {
        // Remove the key if it is not in the mask range.
				s.hashset.Delete(key)
			}
			return false
		})
	}
  // Merge the hashset.
	rs.hashset.Iter(func(key uint64, _ bool) bool {
		s.insertHashValue(key)
		return false
	})
}
```

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Build TopN and Histogram

```plantuml
@startuml



"Analyze Executor" as AE -> AE: sort all samples from the root row sample collector
AE -> AE: send the build task column(index) by column(index)
group concurrently build TopN and Histogram
  AE -> buildWorker1: start the builder worker1 and \nwait for the build task channel
  buildWorker1 -> buildWorker1: build the topN and histogram for the column(index)
  AE -> buildWorker2: start the builder worker2 and \nwait for the build task channel
  buildWorker2 -> buildWorker2: build the topN and histogram for the column(index)
end
AE -> AE: return the analyze result
@enduml
```

<div v-click class="absolute top-30 right-0 transform rotate-30 bg-red-500 text-white font-bold py-1 px-1 rounded-lg"> tidb_build_sampling_stats_concurrency </div>


---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Build TopN

```go {all}{lines:true,maxHeight:'400px'}
// Initialize topNList and cur
topNList := []
cur := samples[0].Value
curCnt := 0
// Iterate through the samples
for sample in samples {
  sampleBytes := sample.Value
  // If the current sample value is the same as cur, increment curCnt
  if sampleBytes == cur {
    curCnt++
    continue
  }
  // If the current sample value is different from cur, handle the count of cur
  if len(topNList) == 0 || curCnt > topNList[-1].Count {
    // Find the position to insert cur
    j := len(topNList)
    for j > 0 && curCnt < topNList[j-1].Count {
      j--
    }
    // Insert cur into topNList
    topNList.insert(j, {Encoded: cur, Count: curCnt})
    // If the length of topNList exceeds numTopN, remove the last element
    if len(topNList) > numTopN {
      topNList = topNList[:numTopN]
    }
  }
  // Update cur and curCnt
  cur = sampleBytes
  curCnt = 1
}
```

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Build Histogram

```go {all}{lines:true,maxHeight:'400px'}
// Initialize variables
sampleNum := int64(len(samples))
sampleFactor := float64(count) / float64(sampleNum)
ndvFactor := float64(count) / float64(ndv)
if ndvFactor > sampleFactor {
    ndvFactor = sampleFactor
}
valuesPerBucket := float64(count)/float64(numBuckets) + sampleFactor
bucketIdx := 0
var lastCount int64
// Append first sample to histogram
hg.AppendBucket(&samples[0].Value, &samples[0].Value, int64(sampleFactor), int64(ndvFactor))
// Iterate through the samples
for i := int64(1); i < sampleNum; i++ {
    upper := new(types.Datum)
    hg.UpperToDatum(bucketIdx, upper)
    cmp, err := upper.Compare(sc.TypeCtx(), &samples[i].Value, collate.GetBinaryCollator())
    if err != nil {
        return 0, errors.Trace(err)
    }
    totalCount := float64(i+1) * sampleFactor
    // Same value as current bucket value
    if cmp == 0 {
        hg.Buckets[bucketIdx].Count = int64(totalCount)
        if hg.Buckets[bucketIdx].Repeat == int64(ndvFactor) {
            hg.Buckets[bucketIdx].Repeat = int64(2 * sampleFactor)
        } else {
            hg.Buckets[bucketIdx].Repeat += int64(sampleFactor)
        }
    } else if totalCount-float64(lastCount) <= valuesPerBucket {
        // Update current bucket
        hg.updateLastBucket(&samples[i].Value, int64(totalCount), int64(ndvFactor), false)
    } else {
        // Store in the next bucket
        lastCount = hg.Buckets[bucketIdx].Count
        bucketIdx++
        hg.AppendBucket(&samples[i].Value, &samples[i].Value, int64(totalCount), int64(ndvFactor))
    }
}
```

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Save Statistics

```plantuml
@startuml



"Analyze Executor" as AE -> AE: wait for analyze results
group concurrently save statistics
AE -> "Stats Save Worker1" as SW1: start the save worker1 and \nwait for the save task channel
SW1 -> SW1: save the analyze result to the system tables
AE -> "Stats Save Worker2" as SW2: start the save worker2 and \nwait for the save task channel
SW2 -> SW2: save the analyze result to the system tables
end
@enduml
```

<div v-click class="absolute top-30 right-0 transform rotate-30 bg-red-500 text-white font-bold py-1 px-1 rounded-lg"> tidb_analyze_partition_concurrency </div>

---
transition: slide-left
---

# Data Structure & Data Flow
TiDB Perspective - Merge Global Statistics

```plantuml
@startuml



"Analyze Executor" as AE -> "Async Global Stats Worker" as AW: create a async global statistics worker
AW -> "IO Worker" as IW: start the IO worker and \nwait for the global statistics task
AW -> "CPU Worker" as CW: start the CPU worker and \nwait for the global statistics task
IW -> IW: load the FMSketches, TopN, and Histogram \nfrom the system tables for each partition
IW -> CW: send FMsketches, TopN, and Histogram \nto the CPU worker through the channel
CW -> CW: receive the FMSketches and \nmerge them to the global FMSketch
group concurrently merge TopN
  CW -> CW: receive the TopN and merge them to the global TopN concurrently
  CW -> "TopN Builder1" as TB1: start the topN builder1 and \nwait for the build task channel
  TB1 -> TB1: merge the topN to the global topN
  CW -> "TopN Builder2" as TB2: start the topN builder2 and \nwait for the build task channel
  TB2 -> TB2: merge the topN to the global topN
end
CW -> CW: receive the Histogram and merge them to the global Histogram
AE -> AE: wait for the IO worker and CPU worker to finish and return the global statistics
AE -> AE: update the global statistics to the system tables
@enduml
```

<div v-click class="absolute top-30 right-0 transform rotate-30 bg-red-500 text-white font-bold py-1 px-1 rounded-lg"> tidb_merge_partition_stats_concurrency </div>


---
transition: slide-up
---

### Nobody can really master TiDB analyze.jpg

<div style="font-size: 11px;">

| Configuration Name                                                                                                         | Description                                                                                                                                                                                                                                                                                                                                          | Default Value | Scope                                                                                                               | Affected Component   |
| :------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------ | :------------------------------------------------------------------------------------------------------------------ | :------------------- |
| <span class="text-red-500" v-mark="{ color: 'red', type: 'underline' }">tidb_build_stats_concurrency</span>                | The number of concurrent workers to analyze <span class="text-red-500" v-mark="{ color: 'red', type: 'underline' }">tables or partitions</span>                                                                                                                                                                                                      | 2             | Global/Session                                                                                                      | TiDB + TiKV          |
| <span class="text-red-500" v-mark="{ color: 'red', type: 'underline' }">tidb_auto_build_stats_concurrency</span>           | The number of concurrent workers to <span class="text-red-500" v-mark="{ color: 'red', type: 'underline' }">automatically analyze tables or partitions</span>                                                                                                                                                                                        | 1             | Global (only for auto analyze)                                                                                      | TiDB (Owner)  + TiKV |
| <span class="text-orange-500" v-mark="{ color: 'orange', type: 'underline' }">tidb_analyze_distsql_scan_concurrency</span> | The number of concurrent workers to <span class="text-orange-500" v-mark="{ color: 'orange', type: 'underline' }">scan regions</span>                                                                                                                                                                                                                | 4             | Global/Session                                                                                                      | TiKV                 |
| tidb_sysproc_scan_concurrency                                                                                              | The number of concurrent workers to scan regions                                                                                                                                                                                                                                                                                                     | 1             | <span class="text-orange-500" v-mark="{ color: 'orange', type: 'underline' }">Global (only for auto analyze)</span> | TiKV                 |
| <span class="text-purple-500" v-mark="{ color: 'purple', type: 'underline' }">tidb_build_sampling_stats_concurrency</span> | 1. The number of concurrent workers to <span class="text-purple-500" v-mark="{ color: 'purple', type: 'underline' }">merge FMSketches and Sample Data</span> from different regions <br/> <br/> 2. The number of concurrent workers to <span class="text-purple-500" v-mark="{ color: 'purple', type: 'underline' }">build TopN and Histogram</span> | 2             | Global/Session                                                                                                      | TiDB                 |
| <span class="text-green-500" v-mark="{ color: 'green', type: 'underline' }">tidb_analyze_partition_concurrency</span>      | The number of concurrent workers to <span class="text-green-500" v-mark="{ color: 'green', type: 'underline' }">save statistics to the system tables</span>                                                                                                                                                                                          | 2             | Global/Session                                                                                                      | TiDB                 |
| <span class="text-blue-500" v-mark="{ color: 'blue', type: 'underline' }">tidb_merge_partition_stats_concurrency </span>   | The number of concurrent workers to <span class="text-blue-500" v-mark="{ color: 'blue', type: 'underline' }">merge global TopN</span>                                                                                                                                                                                                               | 1             | Global/Session                                                                                                      | TiDB                 |

</div>

---
transition: slide-up
layout: center
---

# Q&A

<br/>
<br/>

## Do you have any questions?

---
transition: slide-up
layout: center
---

# Thank You!
