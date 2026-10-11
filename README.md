# DBGuard (Kafei)

> **Enterprise Database Engineering & Static Impact Analysis for Microsoft SQL Server**

DBGuard is an enterprise database engineering platform that helps software engineering teams understand database changes, discover dependencies, identify potential breaking changes, and review risks before database changes reach production.

---

## 1. Core Principles

* **Zero Execution / Pure Static Analysis**: Customer SQL scripts are analyzed strictly as text/AST. DBGuard never executes customer DDL/DML, requires no production database connections, and does not perform automated deployments.
* **Microsoft SQL Server First**: Tailored specifically for T-SQL dialects, table definitions, views, stored procedures, functions, constraints, and indexes.
* **Deterministic & Evidence-Backed**: Every risk finding is linked to explicit source code coordinates (file and line numbers), dependency traversal paths, and concrete rule definitions. No external AI guessing.
* **Multi-Tenant Security**: Strict workspace-level isolation enforced at the backend query and job level.

---

## 2. High-Level Architecture

DBGuard is structured as a decoupled monorepo:

* **`apps/web`**: Next.js frontend (TypeScript, Tailwind CSS) providing visual snapshot comparisons, dependency trees, and risk finding reviews.
* **`apps/api`**: NestJS backend API (TypeScript, Prisma ORM, PostgreSQL) managing users, workspaces, snapshot ingestion, and analysis orchestration.
* **`apps/analyzer`**: Python SQL Analysis Engine (FastAPI, T-SQL AST parser, NetworkX) performing pure static analysis, dependency discovery, diffing, and risk evaluation.
* **`packages/contracts`**: Shared schemas and interface contracts between the API and analyzer.
* **`database/prisma`**: Application database schemas and migrations for PostgreSQL.

---

## 3. Sprint 0.1 Architecture Deliverables

The foundational architecture and requirements documentation created in Sprint 0.1:

* [Product Requirements Document (PRD)](file:///c:/Users/Prabhabi%20Infocom/OneDrive/Desktop/DBGuard/docs/requirements/prd-v1.md): Defines user personas, core 5-stage workflow, V1 vs. deferred scope, and success criteria.
* [System Architecture Document](file:///c:/Users/Prabhabi%20Infocom/OneDrive/Desktop/DBGuard/docs/architecture/system-architecture.md): Component boundaries, sequence diagrams, HTTP interface decisions, and failure recovery.
* [Initial Data Model Specification](file:///c:/Users/Prabhabi%20Infocom/OneDrive/Desktop/DBGuard/docs/architecture/data-model.md): PostgreSQL entity-relationship definitions (ERD), tenant isolation rules, and snapshot immutability.
* [API Design & Contracts](file:///c:/Users/Prabhabi%20Infocom/OneDrive/Desktop/DBGuard/docs/architecture/api-design.md): REST endpoints for web client and internal JSON contract for the Python Analyzer.
* [Threat Model & Security Boundaries](file:///c:/Users/Prabhabi%20Infocom/OneDrive/Desktop/DBGuard/docs/security/threat-model.md): STRIDE threat assessment, path-traversal prevention, resource quotas, and zero-execution guarantees.

---

## 4. Development Status

| Phase | Sprint | Focus | Status |
| :--- | :--- | :--- | :--- |
| **Phase 0** | **Sprint 0.1** | **Product requirements, architecture, data model, API design, threat model** | **Completed** |
| Phase 0 | Sprint 0.2 | Monorepo setup, package configurations, base toolchains | Ready to begin |
| Phase 1 | Sprint 1.x | T-SQL static parser and AST object extractor | Pending |
| Phase 1 | Sprint 2.x | Dependency graph discovery and cycle detection | Pending |
| Phase 1 | Sprint 3.x | Schema snapshot comparison and diff engine | Pending |
| Phase 1 | Sprint 4.x | Deterministic risk rules and impact cascades | Pending |
| Phase 2 | Sprint 5.x | NestJS API, Prisma PostgreSQL schema, and Analyzer client | Pending |
| Phase 3 | Sprint 6.x | Next.js frontend, dashboard, snapshot diffs, and findings UI | Pending |
