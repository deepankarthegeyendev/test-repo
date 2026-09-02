Copilot Sonnet 4.6 Prompt — Implement OpenSearch Search API for GFR .NET 10 Backend

Role

Act as a Principal Software Architect and senior .NET 10 engineer with deep expertise in:

ASP.NET Core .NET 10 Web API

Vertical Slice Architecture

Clean Architecture

CQRS

Amazon OpenSearch Service

DynamoDB

AWS IAM

AWS SDK for .NET

scalable search APIs

filtering, sorting and pagination

enterprise cloud-native APIs

You are modifying an existing enterprise GFR backend.

1. Existing Architecture

The application currently uses DynamoDB as the primary/source-of-truth data store.

There are two relevant DynamoDB tables:

gfr-edg-findings-dev

gfr-edg-rejected-dev

Amazon OpenSearch Service will be introduced for the SEARCH use case.

Target architecture:

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

Important:

DynamoDB remains the source of truth.

OpenSearch is the search/read model.

Search requests must not scan DynamoDB.

Existing remediation/update APIs must continue using DynamoDB where appropriate.

Do not replace existing business functionality merely because OpenSearch is being introduced.

2. Analyze Before Changing Anything

Before modifying files:

Scan the entire workspace.

Identify all .sln, .slnx, and .csproj files.

Identify the API project that should own the Search API.

Identify the current VSA + Clean Architecture structure.

Identify existing:

Controllers

Features

Commands

Queries

Handlers

Services

Interfaces

Infrastructure

DynamoDB repositories

caching

configuration

dependency injection

logging/observability

Check whether an OpenSearch client/package already exists.

Check whether Search functionality already exists.

Inspect the actual attributes/schema used in:

gfr-edg-findings-dev

gfr-edg-rejected-dev

Inspect any existing frontend/API search contract.

Identify existing authentication/authorization and AWS credential conventions.

Do not immediately code.

First provide:

Current architecture assessment.

Existing Search implementation, if any.

Recommended OpenSearch design.

Proposed API contract.

Proposed index strategy.

Proposed field mapping.

Proposed files to create/modify.

Package/dependency changes.

AWS configuration required.

Assumptions and risks.

3. Search Requirements

The Next.js UI needs a search API that can search records originating from both DynamoDB tables.

Support only requirements actually evidenced in the repository/UI.

Evaluate support for:

free-text search

filtering

sorting

pagination

data system

status

finding type

file name

file location

modified date

other existing UI fields

Do not invent unnecessary query parameters.

4. OpenSearch Index Strategy

Evaluate these designs:

Option A — Two indexes

gfr-findings
gfr-rejected

Option B — One unified index

gfr-records

with:

sourceTable

or an equivalent canonical source field.

Choose based on actual schema overlap and UI requirements.

If a unified index is viable, prefer it when it simplifies the Search API and unified UI search.

Do not make the decision just for implementation convenience.

Explain the trade-offs.

5. IMPORTANT: GSI vs OpenSearch Index

Do NOT use a DynamoDB GSI to determine which fields are searchable in OpenSearch.

Keep the concerns separate:

DynamoDB GSI
=
DynamoDB access-path optimization

OpenSearch index
=
Search / filter / sort / read model

A GSI should only be created or changed when an actual DynamoDB access pattern requires it.

Do not create a GSI solely because a field is searchable in OpenSearch.

6. Determine Search Fields

Inspect both DynamoDB tables and produce a field matrix:

Field

OpenSearch Type

Searchable

Filterable

Sortable

Source Table(s)

fileName

text + keyword where appropriate

yes

yes

maybe

inspect

dataSystem

keyword

no

yes

no

inspect

status

keyword

no

yes

no

inspect

modifiedDate

date

no

yes

yes

inspect

Do not assume the sample fields above exist.

Identify the real field names from the code and data models.

Only index attributes needed by the search/read requirements, plus fields required for result display and authorization.

Do not blindly index every DynamoDB attribute.

7. OpenSearch Mapping

Create explicit mappings where needed.

Pay special attention to:

text

keyword

date

numeric

boolean

nested/object fields

Ensure the same logical field has the same OpenSearch type across both source tables.

If two source attributes represent the same business field but have different names, normalize them into one canonical OpenSearch field.

Document every normalization decision.

8. DynamoDB -> OpenSearch Synchronization

Use the existing AWS-native approach where applicable:

DynamoDB
    |
    v
PITR / Streams
    |
    v
OpenSearch Ingestion / Zero-ETL
    |
    v
OpenSearch

The ingestion layer is responsible for:

initial snapshot/backfill

ongoing changes

creates

updates

deletes

required transformations

Do not implement a second custom synchronization mechanism in the .NET API unless the existing architecture explicitly requires it.

If AWS configuration is not present in source control, document exactly what needs to be configured in AWS/IaC, but do not fabricate credentials or infrastructure.

9. Search API Architecture

Implement the API using the existing VSA + Clean Architecture conventions.

Prefer feature organization such as:

Features
└── Search
    └── SearchRecords
        ├── SearchRecordsQuery.cs
        ├── SearchRecordsHandler.cs
        ├── SearchRecordsValidator.cs
        ├── SearchRecordsResponse.cs
        └── ...

Use the repository's established conventions if they differ.

Do not create a generic repository solely for OpenSearch.

10. API Contract

Create or adapt:

GET /api/v1/search

only if that route is consistent with the existing API versioning conventions.

Potential parameters:

query
dataSystem
status
findingType
filterBy
filterValue
sortBy
sortDirection
pageSize
cursor

Expose only parameters justified by the UI/business requirements.

Do not expose raw OpenSearch DSL to the client.

The .NET API translates business-level query parameters into OpenSearch queries.

11. Search Response

Do not expose raw OpenSearch JSON.

Use a stable application-level DTO, for example:

{
  "items": [
    {
      "id": "...",
      "fileName": "...",
      "fileLocation": "...",
      "dataSystem": "...",
      "modifiedDate": "...",
      "status": "...",
      "findingType": "..."
    }
  ],
  "pageSize": 100,
  "nextCursor": "...",
  "hasMore": true
}

Determine whether exact total-count is actually required.

Do not perform expensive count work simply to populate a UI field that does not require it.

12. Pagination

Do not recreate deep DynamoDB offset pagination:

page=9999&pageSize=500

Evaluate the appropriate OpenSearch approach:

search_after

Point in Time (PIT)

cursor abstraction

Prefer a cursor contract to hide OpenSearch-specific details from Next.js.

Example:

GET /api/v1/search?pageSize=100

Response:

{
  "items": [...],
  "nextCursor": "...",
  "hasMore": true
}

Next request:

GET /api/v1/search?pageSize=100&cursor=...

If numbered pages are required by the UI, explicitly explain the scalability trade-off before implementing them.

13. Sorting

Support only an allowlist of sortable fields.

Example:

fileLastModifiedOn
lastUpdateDateUtc
rowCreatedDateOn

Do not inject client-provided field names directly into OpenSearch.

Use a mapping such as:

ModifiedDate
    -> fileLastModifiedOn

Reject unsupported sort fields with the application's standard validation response.

14. Filtering

Use structured filters for exact values.

Examples:

status = Restored
dataSystem = NetApp
findingType = Exception

Prefer OpenSearch keyword fields for exact matching.

Do not implement exact filters as free-text queries.

15. Free-Text Search

Identify which fields participate in the global search box from the existing UI.

Potential examples:

fileName
fileLocation
description
owner

Use an appropriate OpenSearch text-query strategy.

Do not make every field searchable by default.

Avoid unrestricted wildcard queries over huge indexes unless the requirement explicitly demands them.

16. Search Across Both DynamoDB Sources

If using a unified index:

gfr-records

include a source field such as:

sourceTable

Example:

{
  "sourceTable": "gfr-edg-findings-dev"
}

or:

{
  "sourceTable": "gfr-edg-rejected-dev"
}

If using two indexes, create an abstraction so the external API response remains consistent.

Next.js must not need to know how many DynamoDB tables or OpenSearch indexes exist.

17. Authorization

Do not expose OpenSearch directly to the browser.

Required flow:

Next.js
   |
   v
.NET 10 API
   |
   v
OpenSearch

The .NET API is responsible for:

authentication

authorization

tenant/data-system visibility where applicable

allowed filters

allowed fields

Inspect the existing security model before adding new authorization rules.

Do not bypass existing authorization.

18. AWS Authentication

Use the existing AWS credential/identity mechanism.

Do not hard-code:

AWS access keys

secret keys

passwords

tokens

Inspect the runtime environment and use the appropriate IAM role/credential chain.

19. Configuration

Follow the application's existing configuration pattern.

Potential settings:

{
  "OpenSearch": {
    "Endpoint": "...",
    "Index": "gfr-records",
    "RequestTimeoutSeconds": 10
  }
}

Do not put secrets in source-controlled configuration.

Use environment-specific configuration according to existing conventions.

20. Performance

The application already deals with millions of DynamoDB records and may scale significantly.

The Search API must:

query OpenSearch

avoid DynamoDB scans

avoid client-side filtering

avoid client-side sorting

limit page sizes

use source filtering where useful

use cancellation tokens

use bounded request timeouts

avoid unbounded result sets

Return only fields required by the UI.

21. Caching

Inspect the existing caching strategy.

Do not automatically introduce Redis.

If in-memory caching already exists, evaluate whether specific search requests actually have a high enough hit rate to benefit from caching.

If caching is used, ensure the cache key includes every parameter affecting the response.

22. Error Handling

Handle:

OpenSearch timeout

OpenSearch unavailable

malformed query

invalid filter

invalid sort field

invalid cursor

authorization failure

index unavailable

Use the existing application's standard error model.

Do not expose raw OpenSearch exception details to Next.js.

23. Resilience

Reuse any existing Polly/resilience infrastructure.

Use retries carefully.

Search retries must not amplify load during an OpenSearch incident.

Evaluate:

timeout

bounded retry

cancellation

circuit breaker where consistent with platform standards

Do not introduce duplicate resilience stacks.

24. Observability

Use existing structured logging/telemetry.

Capture useful metadata such as:

correlationId
requestId
search duration
page size
filter presence
sort field
result count
OpenSearch response status

Never log secrets, access tokens, or unnecessary sensitive content.

If a common audit/observability framework already exists, reuse it.

25. Testing

Add:

Unit tests

request validation

query construction

field mapping

filter mapping

sort allowlist

cursor generation/decoding

response mapping

error handling

Integration/contract tests

Verify:

free-text search

filters

sorting

pagination

no results

invalid cursor

index unavailable

both source tables/indexes

canonical response format

26. AWS Integration Documentation

Create/update developer documentation explaining:

gfr-edg-findings-dev
        |
        +-------------------+
                            |
gfr-edg-rejected-dev        |
        |                   |
        +-------> OpenSearch Ingestion
                        |
                        v
                 OpenSearch Index
                        |
                        v
                 .NET Search API
                        |
                        v
                    Next.js UI

Document:

Which DynamoDB fields are indexed.

Why each field is indexed.

Which fields are searchable.

Which fields are filterable.

Which fields are sortable.

OpenSearch mappings.

How both DynamoDB tables are represented.

How creates/updates/deletes propagate.

Initial snapshot/backfill strategy.

Authentication/IAM.

Pagination.

Failure behavior.

27. Do Not Over-Engineer

Do not introduce unnecessary:

microservices

repositories

event buses

caches

data stores

abstractions

Use the simplest architecture that satisfies the actual requirements and existing enterprise conventions.

28. Implementation Sequence

Follow this sequence:

Phase 1 — Discovery

Analyze the current repository and existing AWS integration.

Phase 2 — Design

Provide:

index strategy

mapping

searchable/filterable/sortable field matrix

API contract

pagination design

security model

IAM requirements

Phase 3 — One working vertical slice

Implement the Search API end-to-end using one representative source/index.

Phase 4 — Validate

Test search, filters, sorting, pagination and authorization.

Phase 5 — Add second source

Integrate the second DynamoDB source using the same canonical search model.

Phase 6 — Tests and documentation

Add contract/integration tests and document the architecture.

Do not modify all unrelated applications while implementing this feature.

29. Strict Constraints

Do not modify business logic unless required for Search integration.

Do not change existing business DynamoDB table schemas.

Do not create GSI changes solely for OpenSearch searchability.

Do not expose OpenSearch directly to Next.js.

Do not expose raw OpenSearch DSL in the public API.

Do not duplicate search models unnecessarily.

Do not create application-specific search contracts if the UI expects a common result.

Do not create a generic repository solely for OpenSearch.

Do not index every DynamoDB attribute without justification.

Do not introduce a second synchronization mechanism when zero-ETL/OpenSearch Ingestion already provides the required synchronization.

Do not remove or break current DynamoDB functionality.

Do not store secrets in source control.

Do not make unrelated architectural refactoring changes.

Do not change the architectural recommendation merely because I suggest an alternative later. Evaluate the alternative against scalability, security, maintainability, and the requirements, then explain the trade-off.

Once the Search architecture is agreed, treat it as the source of truth unless new evidence demonstrates that it is technically invalid.

Do not implement major architectural changes without first explaining them and showing the impact.

30. Deliverables Before Coding

Before modifying files, provide:

Current Search architecture.

Proposed Search architecture.

One-index vs two-index decision.

Canonical OpenSearch document model.

Searchable/filterable/sortable field matrix.

OpenSearch mappings.

API contract.

Pagination strategy.

IAM/security approach.

AWS ingestion configuration requirements.

Files to create.

Files to modify.

NuGet/dependency changes.

Risks/assumptions.

Then proceed with implementation only after these decisions are clearly documented.

31. Final Acceptance Criteria

The implementation is complete only when:

Next.js calls a .NET 10 Search API.

The Search API queries OpenSearch.

Search does not scan DynamoDB.

Both DynamoDB source tables can be searched through the intended API.

Filtering works for approved fields.

Sorting works for approved fields.

Pagination scales to large result sets.

The API hides OpenSearch implementation details.

DynamoDB remains the source of truth.

OpenSearch is treated as the search/read model.

No unnecessary GSI has been introduced for OpenSearch.

OpenSearch field mappings are explicit where required.

Field types are consistent across both sources.

Only required attributes are indexed.

Authorization is enforced at the API boundary.

AWS authentication uses IAM/approved credential mechanisms.

Unit/integration/contract tests are present.

Documentation explains the complete flow.

Architectural Source of Truth

             BUSINESS DATA
                  |
                  v
              DynamoDB
           SOURCE OF TRUTH
                  |
        Zero-ETL / Ingestion
                  |
                  v
             OpenSearch
          SEARCH READ MODEL
                  |
                  v
            .NET 10 API
        APPLICATION BOUNDARY
                  |
                  v
               Next.js

Do not expose OpenSearch directly to the browser.
