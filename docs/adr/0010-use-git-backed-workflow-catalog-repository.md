# Use Git-Backed Workflow Catalog Repository

The first Workflow Catalog should be maintained in a dedicated Git-backed Workflow Catalog Repository instead of a database and management UI. Workflow Definitions can be stored as declarative files and initially loaded from one simple directory; richer catalog directory governance is deferred. We chose this because the immediate risk is validating definition structure, selector rules, and runtime boundaries; a database-backed management platform and mature catalog governance can be added later after the catalog model stabilizes.
