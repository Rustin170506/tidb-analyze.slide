---
background: "#white"
theme: seriph
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
colorSchema: light
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

<!--
Thanks for joining me today. I'm Rustin, today I'm gonna talk about TiDB Analyze Feature.
It's a feature that collects statistics for the optimizer to generate better query plans based on the cost model.
All my examples and algorithms are based on TiDB v8.1.
If you want to take a look at the source code, you need to make sure you are on the v8.1 tag.
Alright, let's get started.
-->

---
transition: slide-up
---

# Rustin Liu

<div class="leading-8 opacity-80">
PingCAP Optimizer Team Member.<br/>
Cargo/Crates.io/Rustup Maintainer.<br/>
Tokio Console Maintainer.<br/>
</div>

<div my-10 grid="~ cols-[40px_1fr] gap-y4" items-center justify-center>
  <div i-ri-github-line op50 ma text-xl/>
  <div><a href="https://github.com/Rustin170506" target="_blank">Rustin170506</a></div>
  <div i-ri-firefox-line op50 ma text-xl/>
  <div><a href="https://rustin.me" target="_blank">rustin.me</a></div>
</div>

<img src="https://avatars.githubusercontent.com/u/29879298?v=4" rounded-full w-30 abs-tr mt-22 mr-22/>

<div flex="~ gap2">
</div>

<!--
Let me introduce myself.

I just transferred to the optimizer team about a year ago.

I'm also very active in the Rust community. If you have any questions about the Rust toolchain, feel free to reach out to me.
We can discuss it together.

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

<!--

This is today's agenda.

We'll start with an overview of the Analyze feature with some examples.

Then we will dive into the data structure and data flow overview.
We will look at the data structure and data flow from both the TiKV and TiDB perspectives.
Finally, we will have a Q&A session. If you have any questions, feel free to send it to the chat.

And I will try to answer them in the Q&A session. If I can't answer them, maybe kunqin, yiding and other team members can help me.

-->

---
transition: slide-left
layout: center
---

# Analyze Overview


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

<!--

After executing the Analyze statement, we’ll generate some statistics from the TiKV.

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

<!--

Let’s start with a simple example.

First, we’ll create a table named t with a single column a.

Next, we’ll insert 2000 rows into the table.

Notice that we insert the same value twice for every even number.

Keep this example in mind, because we’ll use it later to demonstrate the data structure.

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

<!--

Let’s look at a simple query that selects rows where column A equals 100.

This is a basic example of column selectivity, using a simple equality condition. The optimizer estimates that there are 2 rows that match this condition.

It uses the TopN data structure to make this estimate.

The value 100 is in the TopN data structure, because it’s an even number and we inserted it twice.

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


<!--

Under the hood, we use a system table called stats_top_n to store the TopN data structure.

If we query this table, we can see the top N values and their counts.

For example, we can see that the first value 0x0380000000000000 is in the TopN data structure with a count of 2.

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

<!--

Now, let’s look at another example.

This time, we’re selecting rows where column A equals 1999.

Since we only collect the first top 500 values, this value isn’t in the TopN data structure.

In this case, the optimizer uses the Histogram data structure to estimate the number of matching rows.

So, in this example, the optimizer estimates that only 1 row matches the condition. The reason is that the value 1999 is in the Histogram data structure. I will show you the Histogram data structure in the next slide. So we littery hint this value in the histogram.

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


<!--
Same as before, we use another system table called stats_buckets to store the Histogram data structure.

If we query this table, we can see the histogram buckets.

In this example, we can see that the value 1999 is in a histogram bucket with a count of 1.

The lower and upper bounds of this bucket are both 1999.

If we look at the next bucket, we see it has a count of 9, with a lower bound of 1993.

Additionally, the repeats are 2, meaning there are two repeated values of the upper bound.

We can use this repeated value to estimate the selectivity of the column for an equality condition. Is is kind of like a special TopN in the histogram. But it is only for the upper bound.

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
- NDV: The number of distinct values in the bucket.(**Deprecated, always 0**)

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

<!--

Let’s take a closer look at the histogram bucket.

Just the ID of the bucket.

The lower bound of the bucket.

The upper bound of the bucket.

The number of values in the bucket. This is a cumulative count, including all values up to the bucket. But when we store the data, we only store the count of the bucket.
During we load the histogram, we will calculate the cumulative count.

The number of repeated values at the upper bound.

 The number of distinct values in the bucket. But this field is deprecated and always 0. Because we use the sample data to build the histogram, so we don't really know the accurate NDV.

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

<!--

As you can see, we use a equi-height histogram to store the data.

The reason we use an equi-height histogram is that it provides a more accurate estimation of the selectivity of the column compared to an equi-width histogram.

If you want to know why it is more accurate, you can find more information in the paper linked in the footnotes.

The whole paper is about how to estimate the number of tuples satisfying a condition with a equi-height histogram.

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

<!--

Now, let’s look at the last example.

This time, we’re selecting rows where column A equals 9999. Since the value 9999 isn’t in the TopN data structure or the Histogram data structure, the optimizer uses a different method to estimate the number of matching rows.

In this case, the optimizer

The optimizer uses the NotNullCount and NDV to make this estimate.

So let me explain what is the NDV and NotNullCount.

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


<!--

The NotNullCount is pretty straightforward. It is the number of not null values in the column.

The NDV is the number of distinct values in the column.

The not null count is easy to calculate, but the NDV is a bit more complicated.

To calculate the NDV, we use the Flajolet-Martin Sketch algorithm. This algorithm provides an efficient way to estimate the number of distinct values in a large dataset.

Before we dive into the details of the algorithm, let’s take a look at the data flow from the big picture.

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

<!--

This is the data flow overview of the Analyze feature.

First, the TiDB owner or client executes the Analyze statement.

Next, TiDB sends an analyze gRPC request to TiKV to collect statistics.

After collecting the statistics, TiKV sends an analyze gRPC response back to TiDB.

Finally, TiDB builds and merges the statistics and updates the system tables as we discussed earlier.

We use the same coprocessor framework to collect statistics. This is a special request handled by the coprocessor.

I won’t go into the details of the coprocessor framework in this talk, but you can find more information in the TiKV repository.

-->

---
transition: slide-up
layout: center
---

# Data Structure & Data Flow
TiKV Perspective

In TiKV, we only do two things:

1. Calculate the FMSketch.
2. Sample the data.

<!--

In TiKV, we only do two things:

1.	Calculate the FMSketch: We scan all the data and calculate the FMSketch to estimate the NDV. Note that this is the most expensive part of the process because we need to scan all the data.
2.	Sample the data: We cannot send all the data to TiDB, so we need to sample it.

-->


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

<!--

Before we dive into the details of the FMSketch implementation, let’s look at the mathematical assumptions.

First, we assume that the hash functions are independent. This means that the hash function h(x) uniformly distributes input elements over a large range of integers.

Second, we assume that the number of trailing zeros in the binary representation of the hash values follows a geometric distribution. In other words, the number of trailing zeros is fairly random.

These assumptions are crucial for the accuracy of the FMSketch.

If you want to learn more about the FMSketch algorithm, you can find the original paper in the footnotes.

-->

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

<!--
The FMSketch algorithm is based on three main steps:

1. We map each element of the set to an integer using the hash function h(x).
2. For each hash value, we count the number of trailing zeros in its binary representation. We record the maximum count R.
3. To estimate the cardinality of the set, we use the maximum trailing zero count R with the formula 2 to the power of R.
-->

---
transition: slide-left
---

<FMSketch/>

<!--
This is a more detailed example of the FMSketch algorithm.

After we click the generate button to see the hash values and trailing zeros for each element.

You’ll see that the maximum trailing zero count is 4, and the estimated cardinality is 16.

This is a perfect example, as the estimated cardinality matches the actual cardinality of the set.

However, in practice, there will be some cases where the estimation is not as accurate.
-->

---
transition: slide-left
---

<BadFMSketch/>

<!--
This time, we have an outlier in the set.

The hash value of the outlier is jj, which has 6 trailing zeros.

The maximum trailing zero count is 6, and the estimated cardinality is 64.

But the actual cardinality of the set is 16.

This is an example where the FMSketch algorithm doesn’t provide an accurate estimation.

To address this issue, we need to use multiple hash functions and take the median estimate.

However, using multiple hash functions increases the cost. Because we need to scan the data multiple times and calculate the hash value multiple times.

So, we chose another algorithm called Distinct Sampling to solve this problem.
-->

---
transition: slide-left
---

# Data Structure - Distinct Sampling
TiKV Perspective

Core Principles [^1]
1. **Hash Function:**
   - Use a hash function that maps each distinct value to a random `die-level`.
2. **Sample Maintenance:**
   - Maintain a sample S of distinct values and a current level `l`.
3. **Sampling Criterion:**
   - Keep values in S only if their `die-level ≥ l`.
4. **Cardinality Estimation:**
   - Estimate distinct items as $|S| * 2^l$

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

<!--
To address the limitations of the FMSketch algorithm, we use a new algorithm called Distinct Sampling.

The core principles of Distinct Sampling are as follows:

Use a hash function that maps each distinct value to a random die-level. This is very similar to the FMSketch algorithm.

Maintain a sample S of distinct values and a current level L.
Keep values in S only if their die-level is greater than or equal to L.

Estimate the number of distinct items as the size of the sample S multiplied by 2 to the power of L.

We use a hash function to map each distinct value to a random value and check its binary representation to determine the die-level. If there are no trailing zeros, the die-level is 0. If there is one trailing zero, the die-level is 1, and so on.
-->

---
transition: slide-left
---

# Data Structure - Distinct Sampling
TiKV Perspective

Algorithm Steps
1. **Initialization:**
   - Start with l = 0 and an empty sample S.
2. **Processing Each Row:**
   - For each row r with target attribute value v:
     - Compute die-level = $h(v)$
     - If die-level ≥ l:
       - Add r to S
3. **Sample Size Control:**
   - If $|S| > k$, increment l and remove items with die-level < l.

<!--
The Distinct Sampling algorithm consists of three main steps:

Start with l = 0 and an empty sample S.

For each row r with the target attribute value v:
Compute the die-level using the hash function h(v).
If the die-level is greater than or equal to l, add r to the sample S.

If the size of the sample S is greater than k, increment L and remove items with a die-level less than L.

This is the sampling part of the algorithm. We only keep the items with a die-level greater than or equal to L.

That means for the lower die-level, we have enough samples to prove that the values follow this pattern.
So, a bigger sample size means we can get a more accurate estimation. This is the key point in solving the problem of FMSketch.
-->

---
transition: slide-left
---

<FMSketchDemo/>

<!--
This is a more detailed example of the Distinct Sampling algorithm.

Every time you click the “Next” button, we will process a new row.

The sample size is set to 8, so we will increment the die-level when the sample size is bigger than 8.

As you can see, even with an outlier in the set, the estimated cardinality remains accurate.
-->

---
transition: slide-left
---

# Data Structure - Distinct Sampling
TiKV Perspective

Estimation and Analysis

1. **Accuracy:**
   - Provides estimates within 0%-10% relative error
   - Much more accurate than previous sampling methods
2. **Efficiency:**
   - Single pass over the data.
   - Only one hash function required.

<!--

The Distinct Sampling algorithm provides a more accurate estimation of the number of distinct items in a set.

The algorithm provides estimates within a 0% to 10% relative error, making it much more accurate than previous FMSketch algorithms.

The algorithm is also efficient, as it only requires a single pass over the data and one hash function.

-->


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

<!--
The Bernoulli Sampling algorithm is the method used to sample data.

The algorithm is based on the following mathematical assumptions:

Each sample in the data set is selected independently from other samples.
Each sample is selected with a fixed probability p (0 ≤ p ≤ 1), uniformly across the entire data set.

Each sample selection follows a Bernoulli distribution with parameter p.
-->

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

<!--
The Bernoulli Sampling algorithm is based on the following septs:

Define a sampling probability p for selecting each sample.

For each sample in the data set, generate a random number and compare it to p.

If the random number is less than p, include the sample in the resulting subset.
-->

---
transition: slide-left
---

<BernoulliSampling/>

<!--

This is a more detailed example of the Bernoulli Sampling algorithm.

After we click the generate button we will see the Bernoulli Sampling result.

It is pretty straightforward. We generate a random number for each sample and compare it to the sampling probability.

-->

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

<!--

After we calculate the FMSketches and sample the data in TiKV, we send the results back to TiDB.

In TiDB, we perform the following steps:

1. Merge all FMSketches and Sample Data. Because it is a distributed system, we need to merge the results from all TiKV nodes.
2. Build the TopN and Histogram.
3. Store the statistics in the system tables.

-->

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

<!--

We have already seen the data flow from the TiKV perspective.

Now, let’s look at the data flow from the TiDB perspective.

-->


---
transition: slide-left
---

### 🤡Nobody can really master TiDB analyze.jpg

<div style="font-size: 11px;">

| Configuration Name                                                                                                                | Description                                                                                                                                                                                                                                                                                                                                                        | Default Value | Scope                                                                                                                      | Affected Component   |
| :-------------------------------------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------ | :------------------------------------------------------------------------------------------------------------------------- | :------------------- |
| <span class="text-red-500" v-mark="{ at: 1, color: 'red', type: 'underline' }">tidb_build_stats_concurrency</span>                | The number of concurrent workers to analyze <span class="text-red-500" v-mark="{ at: 1, color: 'red', type: 'underline' }">tables or partitions</span>                                                                                                                                                                                                             | 2             | Global/Session                                                                                                             | TiDB + TiKV          |
| <span class="text-red-500" v-mark="{ at: 1, color: 'red', type: 'underline' }">tidb_auto_build_stats_concurrency</span>           | The number of concurrent workers to <span class="text-red-500" v-mark="{ at: 1, color: 'red', type: 'underline' }">automatically analyze tables or partitions</span>                                                                                                                                                                                               | 1             | Global (only for auto analyze)                                                                                             | TiDB (Owner)  + TiKV |
| <span class="text-orange-500" v-mark="{ at: 1, color: 'orange', type: 'underline' }">tidb_analyze_distsql_scan_concurrency</span> | The number of concurrent workers to <span class="text-orange-500" v-mark="{ at: 1, color: 'orange', type: 'underline' }">scan regions</span>                                                                                                                                                                                                                       | 4             | Global/Session                                                                                                             | TiKV                 |
| tidb_sysproc_scan_concurrency                                                                                                     | The number of concurrent workers to scan regions                                                                                                                                                                                                                                                                                                                   | 1             | <span class="text-orange-500" v-mark="{ at: 1, color: 'orange', type: 'underline' }">Global (only for auto analyze)</span> | TiKV                 |
| <span class="text-purple-500" v-mark="{ at: 1, color: 'purple', type: 'underline' }">tidb_build_sampling_stats_concurrency</span> | 1. The number of concurrent workers to <span class="text-purple-500" v-mark="{ at: 1, color: 'purple', type: 'underline' }">merge FMSketches and Sample Data</span> from different regions <br/> <br/> 2. The number of concurrent workers to <span class="text-purple-500" v-mark="{ at: 1, color: 'purple', type: 'underline' }">build TopN and Histogram</span> | 2             | Global/Session                                                                                                             | TiDB                 |
| <span class="text-green-500" v-mark="{ at: 1, color: 'green', type: 'underline' }">tidb_analyze_partition_concurrency</span>      | The number of concurrent workers to <span class="text-green-500" v-mark="{ at: 1, color: 'green', type: 'underline' }">save statistics to the system tables</span>                                                                                                                                                                                                 | 2             | Global/Session                                                                                                             | TiDB                 |
| <span class="text-blue-500" v-mark="{ at: 1, color: 'blue', type: 'underline' }">tidb_merge_partition_stats_concurrency </span>   | The number of concurrent workers to <span class="text-blue-500" v-mark="{ at: 1, color: 'blue', type: 'underline' }">merge global TopN</span>                                                                                                                                                                                                                      | 1             | Global/Session                                                                                                             | TiDB                 |

</div>


<!--

But I don't really have enough confidence to say that I can really teach you how to understand the Analyze feature.

Because there are so many configuration settings that control how the Analyze feature runs concurrently in TiDB and TiKV. And the observability is not that good. Nobody can really master it I believe.

I think there’s definitely a big room for improvement. Some names are so bad that nobody can really get a clear idea of what they do.

So don’t worry if you don’t understand everything in my following slides. I don’t understand everything either.

I’ll try my best to explain the details of the Analyze feature.

But even for me, it took a while to understand how everything works together.

So let’s dive into the details.

-->

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

<!--
This is the data flow for analyzing tables or partitions concurrently in TiDB.

We analyze tables or partitions concurrently to improve the performance of the Analyze feature. For partitioned tables, we may have many partitions, and analyzing them concurrently can significantly reduce the overall time.

Here’s how it works:

1. Create tasks for each table or partition. It just a normal query so we use the same framework to build the plan.
2. Start the tasks for each table and wait for their completion.
3. Once the analysis is done, update the statistics to the system tables in the result handler.
4. For partitioned tables, merge the statistics from each partition to get the global statistics.
5. Load the latest statistics into the cache to ensure the query optimizer uses the most up-to-date statistics.


The concurrency of the this part is controlled by the tidb_build_stats_concurrency variable. By default, the concurrency is set to  2
-->

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

<!--
Let’s dive into the details of the data flow for scanning regions concurrently in TiDB.

When we analyze a table, we need to scan all the regions of the table to collect statistics. To improve the performance, we scan regions concurrently. Here’s how it works:

1. Open the Analyze Executor and Initialize the analyze process.
2. Start the Coprocessor Workers: and Start the tasks for different ranges and wait for their completion.
3. Send an analyze gRPC request to each region to scan and collect statistics.
4. Once the analysis is done, build and merge the statistics and return the analyze result.

To control the concurrency of scanning regions, you can adjust the tidb_analyze_distsql_scan_concurrency variable. By default, the concurrency is set to 4.
-->

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

<!--
After receiving all the FMSketches and sample data from TiKV, we need to merge them in TiDB.
To enhance the performance, we merge the FMSketches and sample data concurrently.

Here’s how it works:
1.  Initialize the root row sample collector.
2. Start the tasks and wait for their completion.
3. In each worker,  it merges the FMSketches and sample data into the root row sample collector.
4. Once the merging is complete, build the TopN and Histogram, and return the analyze result.

To control the concurrency of merging FMSketches and sample data, you can adjust the tidb_build_sampling_stats_concurrency variable. By default, the concurrency is set to 2.
-->

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

<!--
After merging the FMSketches and sample data, our next step is to build the TopN and Histogram data structures in TiDB.

To enhance the performance, again, we construct the TopN and Histogram concurrently on a column or index basis.  Here’s the process:

1.  First, we sort all the samples collected in the root row sample collector.
2.  Next, we dispatch build tasks for each column or index.
3.  We then initiate the builder workers to start these tasks and wait for their completion.
4. For each column or index, we build the TopN and Histogram.  Do some calculation and build the data structure.
5. Finally, once the building is complete, we return the analyze result.

To manage the concurrency of building the TopN and Histogram, you can adjust the tidb_build_sampling_stats_concurrency variable. By default, it’s set to 2.
-->

---
transition: slide-left
layout: center
---

<TopNAnimation />


---
transition: slide-left
layout: center
---

<HistogramAnimation />

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

<!--

Finally, after building the TopN and Histogram data structures, we save the statistics to the system tables in TiDB.

To enhance the performance, again, we save the statistics concurrently. Here’s how it works:

1. First, we wait for the analyze results to be ready.
2. We then start the save workers and wait for the save task channel.
3. Each save worker saves the analyze result to the system tables concurrently.

The concurrency of saving statistics is controlled by the tidb_analyze_partition_concurrency variable. By default, the concurrency is set to 2.

-->

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

<!--

For partitioned tables, we need to merge the statistics from each partition to build the global statistics. To enhance the performance, again, we merge the global statistics concurrently in TiDB. Here’s how it works:

1. Initialize the async global statistics worker.
2. Start the IO Worker: Load the FMSketches, TopN, and Histogram from the system tables for each partition.
3. Start the CPU Worker: Send the FMSketches, TopN, and Histogram to the CPU worker through the channel.
4. Merge FMSketches: Receive the FMSketches and merge them into the global FMSketch.
5. Merge TopN: Receive the TopN and merge them into the global TopN concurrently.
6. Merge Histogram: Receive the Histogram and merge them into the global Histogram.
7. Wait for the IO worker and CPU worker to finish.
8. Update the global statistics in the system tables.

The concurrency of merging global statistics is controlled by the tidb_merge_partition_stats_concurrency variable. By default, the concurrency is set to 1.

-->

---
transition: slide-up
---

# Improve the Analyze
What can we do?

<div class="container">
 <div class="tracking">
    <h3>Tracking Document</h3>
    <p><a href="https://github.com/pingcap/tidb/issues/55043">Statistics Tech Debt</a></p>
  </div>
  <br/>
  <div class="blogs">
    <h3>Blogs</h3>
    <p>
      I started a new series blog post to discuss the Analyze feature improvement.
    </p>
      <p><a href="https://pingcap-cn.feishu.cn/wiki/S1zuwQmDwigxhBkDgEVcVbXen4c">NCRMTA1: Surprise analyze-partition-concurrency-quota</a></p>
      <p><a href="https://pingcap-cn.feishu.cn/docx/I6tRd97W9o5ciux8yZIcRyFOnYe">NCRMTA2: Accelerate Auto-Analyze of Partitioned Tables</a></p>
  </div>
</div>

<!--
The Analyze feature is a critical part of the TiDB query optimizer. It is the only input for the query optimizer to generate the better query plan right now.

As you can see, this module is very complex and have a lot of issues.

If you are interested in the Analyze feature, you can track the progress of the project in the tracking document.

And I just started a new series of blog posts to discuss the Analyze feature improvement. Hope we can make it better in the future.

Aright! That’s all for my talk today. Thank you for listening. Do you have any questions?
-->

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
