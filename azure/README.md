# Azure learning notes

Notes from exploring Microsoft Azure. Each topic has its own `.md` file; this file is the
index.

## Access control & Azure AI

- [Using Azure AI only on data a user is allowed to see](ai-permission-aware-data-access.md) —
  how to guarantee Azure AI only processes database rows / SharePoint documents (and
  Databricks tables) that the signed-in user's role permits, using Entra ID + RLS +
  document-level access control, before the model ever sees the data.

## Azure Databricks

- [What is the role of Databricks? A lakehouse, not just a warehouse or a wrapper](databricks-role-lakehouse.md) —
  Databricks as a lakehouse: separates compute from open-format Delta storage in ADLS; can be
  a warehouse (clone + massage, medallion architecture) **and** a compute / orchestration
  wrapper over external data.
