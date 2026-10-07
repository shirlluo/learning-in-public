# Databricks: Intro & Compute

The core building blocks of the Databricks platform: where data and code live, how governance works, and what actually runs your queries.

---

## 1. The workspace at a glance

- **Catalog**: all the available data assets in one place, the one-stop shop for data management.
- **Workspace**: all the code you've been working on.
- **Notebooks**: based on Jupyter Notebooks, and a single notebook can mix multiple languages.

**Supported languages**

|Language|Typical use|
|---|---|
|Scala|data engineering|
|Python|general purpose, data engineering, data science|
|SQL|BI and data engineering|
|R|data science|

---

## 2. Unity Catalog

Unity Catalog is the governance layer on top of everything in the lakehouse.

- Provides a single, holistic governance layer.
- Provides granular access control for every data asset, from data tables to ML models.

---

## 3. Compute

A **cluster** is a collection of resources that process data and perform analytics.

### Classic vs. serverless

||Classic|Serverless|
|---|---|---|
|How it's created|resources are requested from your cloud provider|resources are created in Databricks' control plane|
|Control|full control, all reuse existing cloud resources|access granted to users through Unity Catalog|
|Pros|compute and security stay in your own environment|latest features, faster performance|
|Cons|slow startup, since every resource is created from scratch|less control over the underlying infrastructure|

### Single-node vs. multi-node

A single-node cluster runs everything on one machine, while a multi-node cluster distributes work across several.

### Databricks Runtime

The runtime version determines which libraries and features are available. The recommended choice is **LTS (Long-Term Support)**.

---

## Quick reference: which do I need?

- **Browse every available data asset**: Catalog
- **Find the code I've been working on**: Workspace
- **Control who can see which tables or ML models**: Unity Catalog
- **Need compute and security inside my own cloud environment**: classic cluster
- **Need fast startup and the newest features**: serverless
- **Not sure which runtime version to pick**: LTS

---

_Part of my [Databricks notes](https://claude.ai/chat/README.md), written while upskilling in data analytics._
