# Implement OpenSearch Search API for GFR .NET 10 Backend

## Role

Act as a Principal Software Architect and senior .NET 10 engineer with deep expertise in:

- ASP.NET Core .NET 10 Web API
- Vertical Slice Architecture
- Clean Architecture
- CQRS
- Amazon OpenSearch Service
- DynamoDB
- AWS IAM
- AWS SDK for .NET
- enterprise search APIs
- scalable pagination/filtering/sorting
- secure cloud-native APIs

You are modifying an existing enterprise GFR backend.

---

# 1. Existing Architecture

The application currently uses DynamoDB as the primary/source-of-truth data store.

There are two relevant DynamoDB tables:

1. `gfr-edg-findings-dev`
2. `gfr-edg-rejected-dev`

Amazon OpenSearch Service will be introduced for the SEARCH use case.

The intended architecture is:

```text
Next.js UI
    |
    v
.NET 10 Search API
    |
    v
Amazon OpenSearch Service
    ^
    |
DynamoDB -> OpenSearch Ingestion / Zero-ETL
