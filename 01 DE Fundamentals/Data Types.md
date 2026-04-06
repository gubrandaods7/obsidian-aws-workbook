
## 1. Overview
High-level definition of data classification based on structure.

## 2. Types
### Structured
- Tabular format
- Schema enforced
- Examples: SQL databases

### Semi-Structured
- Flexible schema
- JSON, XML

### Unstructured
- No predefined schema
- Media, text

## 3. AWS Mapping
| Type | Services |
|------|--------|
| Structured | RDS, Redshift |
| Semi | DynamoDB, S3 |
| Unstructured | S3 |

## 4. Trade-offs
- Structured → performance, rigid
- Semi → flexible, moderate query complexity
- Unstructured → scalable, hard to query

## 5. When to Use
- Structured → analytics, BI
- Semi → APIs, logs
- Unstructured → raw ingestion

## 6. Pitfalls
- Treating JSON as fully unstructured
- Overusing DynamoDB for relational use cases

## 7. Related Concepts
- [[data-lake]]
- [[schema-evolution]]

## 8. Exam Notes
- JSON = semi-structured (IMPORTANT)