---
theme: seriph
background: https://github.com/pingcap/tidb/assets/29879298/b565d5ea-902a-4082-b7b7-1ebce80ba029
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
mdc: true
colorSchema: dark
---

# TiDB Analyze

A Deep Dive

Based on TiDB [v7.6.0](https://github.com/pingcap/tidb/tree/v7.6.0)

RUSTIN LIU

<div class="pt-12">
  <span @click="$slidev.nav.next" class="px-2 py-1 rounded cursor-pointer" hover="bg-white bg-opacity-10">
    Begin <carbon:arrow-right class="inline"/>
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

---
transition: slide-up
---

# Rustin Liu

<div class="leading-8 opacity-80">
PingCAP Database Engineer.<br/>
Cargo Contributor.<br/>
Crates.io Maintainer.<br/>
Rustup Previous Maintainer.<br/>
</div>

<div my-10 grid="~ cols-[40px_1fr] gap-y4" items-center justify-center>
  <div i-ri-github-line op50 ma text-xl/>
  <div><a href="https://github.com/hi-rustin" target="_blank">hi-rustin</a></div>
  <div i-ri-twitter-line op50 ma text-xl/>
  <div><a href="https://twitter.com/hi_rustin" target="_blank">hi_rustin</a></div>
  <div i-ri-firefox-line op50 ma text-xl/>
  <div><a href="https://hi-rustin.rs" target="_blank">hi-rustin.rs</a></div>
  <div i-ri-youtube-line op50 ma text-xl/>
  <div><a href="https://www.youtube.com/@hi-rustin" target="_blank">hi-rustin</a></div>
</div>

<img src="https://avatars.githubusercontent.com/u/29879298?v=4" rounded-full w-30 abs-tr mt-22 mr-22/>

<div flex="~ gap2">
</div>

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
  - Data Structure & Data Flow (TiKV Perspective)
  - Data Structure & Data Flow (TiDB Perspective)
  - Q&A

  </div>
</div>

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

---
transition: slide-left
---

# Analyze Statement


```sql
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
transition: slide-left
---

# Analyze Options

<div class="flex flex-col justify-center items-center h-50">
<p>
ANALYZE TABLE t PARTITION p1 <span v-mark.circle.orange>COLUMNS c1, c2</span>;
</p>
<p>
ANALYZE TABLE t COLUMNS c1, c2 <span v-mark.circle.orange>WITH 20 TOPN</span>;
</p>
</div>

<div v-click class="text-xl text-center">
We will store these options in the system tables.
In the future, we can use these options to optimize the analyze process.
</div>

<br/>
<br/>

<div v-click class="text-xl text-center">
<span v-mark="{ at: 4, color: 'orange', type: 'underline' }">ANALYZE TABLE t;</span>
</div>

---
transition: slide-up
---

# Data Flow

<br/>

<div class="flex justify-center items-center">

```plantuml
@startuml

skinparam monochrome reverse

"TiDB Owner/Client" as TC -> TiDB: execute analyze statement
TiDB -> TiKV: send analyze gRPC request
TiKV -> TiKV: collect statistics
TiKV --> TiDB: send analyze gRPC response
TiDB -> TiDB: update statistics to system tables
TiDB --> TC: return success
@enduml
```
</div>

---
transition: slide-up
---

# Protocol Buffers - kvproto
It is similar to a SELECT statement. We send a coprocessor request to TiKV to collect statistics.

## Request

```proto
message Request {
    kvrpcpb.Context context = 1;
    int64 tp = 2;
    bytes data = 3;
    uint64 start_ts = 7;
    repeated KeyRange ranges = 4;
}
```

## Response

```proto
message Response {
    bytes data = 1;
    errorpb.Error region_error = 2;
    kvrpcpb.LockInfo locked = 3;
    string other_error = 4;
    KeyRange range = 5;
}
```

---
transition: slide-up
---

# Protocol Buffers - tipb

## Request

```proto
enum AnalyzeType {
    TypeIndex = 0;
    TypeColumn = 1;
    TypeCommonHandle = 2;
    TypeSampleIndex = 3;
    TypeMixed = 4;
    TypeFullSampling = 5;
}

message AnalyzeReq {
    optional AnalyzeType tp = 1;
    optional uint64 flags = 3;
    optional int64 time_zone_offset = 4;
    optional AnalyzeIndexReq idx_req = 5;
    optional AnalyzeColumnsReq col_req = 6;
}
```

---
transition: slide-up
---

# Protocol Buffers - AnalyzeColumnsReq

## Request

```proto
message AnalyzeColumnsReq {
    optional int64 bucket_size = 1;
    optional int64 sample_size = 2;
    optional int64 sketch_size = 3;
    repeated ColumnInfo columns_info = 4;
    optional int32 cmsketch_depth = 5;
    optional int32 cmsketch_width = 6;
    repeated int64 primary_column_ids = 7;
    optional int32 version = 8;
    repeated int64 primary_prefix_column_ids = 9;
    repeated AnalyzeColumnGroup column_groups = 10;
    optional double sample_rate = 11;
}
```

---
transition: slide-up
---

# Protocol Buffers - AnalyzeColumnsResp

## Response

```proto
message SampleCollector {
    repeated bytes samples = 1;
    optional int64 null_count = 2;
    optional int64 count = 3;
    optional FMSketch fm_sketch = 4;
    optional CMSketch cm_sketch = 5;
    optional int64 total_size = 6;
}

message RowSampleCollector {
    repeated RowSample samples = 1;
    repeated int64 null_counts = 2;
    optional int64 count = 3;
    repeated FMSketch fm_sketch = 4;
    repeated int64 total_size = 5;
}

message AnalyzeColumnsResp {
    repeated SampleCollector collectors = 1;
    optional Histogram pk_hist = 2;
    optional RowSampleCollector row_collector = 3;
}
```
