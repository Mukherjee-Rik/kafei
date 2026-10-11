# API Design & Interface Contracts — DBGuard

## 1. Overview

DBGuard maintains two distinct API boundaries:
1. **Public/Client API**: REST interface exposed by the NestJS backend to the Next.js web application (and future CLI/CI tools).
2. **Internal Analysis Contract**: Versioned HTTP/JSON interface between the NestJS backend and the Python SQL Analysis Engine.

All APIs use standard HTTP response codes, JSON request and response payloads, and uniform error formats.

---

## 2. Public / Client API (NestJS Backend)

### 2.1 Standard Response & Error Envelopes

#### Success Envelope
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2026-10-11T17:15:00.000Z",
    "requestId": "req-98f2a1b7-e21a"
  }
}
```

#### Error Envelope
```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "The requested snapshot does not exist in this workspace",
    "details": []
  },
  "meta": {
    "timestamp": "2026-10-11T17:15:00.000Z",
    "requestId": "req-98f2a1b7-e21a"
  }
}
```

### 2.2 Core Endpoint Groups

#### Authentication & Profile (`/api/v1/auth`)
* `POST /api/v1/auth/register` — Register initial user and default workspace.
* `POST /api/v1/auth/login` — Authenticate and receive JWT access token.
* `GET /api/v1/auth/me` — Retrieve current authenticated user profile and memberships.

#### Workspace Management (`/api/v1/workspaces`)
* `GET /api/v1/workspaces` — List workspaces accessible to current user.
* `POST /api/v1/workspaces` — Create a new workspace.
* `GET /api/v1/workspaces/:workspaceId` — Retrieve workspace details.
* `GET /api/v1/workspaces/:workspaceId/members` — List members and roles.
* `POST /api/v1/workspaces/:workspaceId/members` — Invite or add member.

#### Project Management (`/api/v1/workspaces/:workspaceId/projects`)
* `GET /api/v1/workspaces/:workspaceId/projects` — List projects in workspace.
* `POST /api/v1/workspaces/:workspaceId/projects` — Create project.
* `GET /api/v1/workspaces/:workspaceId/projects/:projectId` — Project overview and snapshot counts.

#### Snapshots (`/api/v1/workspaces/:workspaceId/projects/:projectId/snapshots`)
* `GET .../snapshots` — List snapshots for project.
* `POST .../snapshots` — Ingest a new snapshot (multipart upload of `.zip` or `.sql` files, or commit reference).
* `GET .../snapshots/:snapshotId` — Snapshot status, file count, and object summary.
* `GET .../snapshots/:snapshotId/objects` — Filterable list of parsed database objects (tables, views, procs).
* `GET .../snapshots/:snapshotId/dependencies` — Dependency graph nodes and edges for snapshot.

#### Comparisons & Analysis (`/api/v1/workspaces/:workspaceId/projects/:projectId/comparisons`)
* `POST .../comparisons` — Trigger comparison between `baseSnapshotId` and `targetSnapshotId`.
  * Request Body:
    ```json
    {
      "baseSnapshotId": "uuid-base",
      "targetSnapshotId": "uuid-target",
      "options": { "includeFormattingChanges": false }
    }
    ```
* `GET .../comparisons/:comparisonId` — Comparison status, summary of additions/deletions/modifications, and risk counts.
* `GET .../comparisons/:comparisonId/findings` — Paginated risk findings filtered by severity (`CRITICAL`, `HIGH`, etc.).
* `GET .../comparisons/:comparisonId/report` — Exportable structured report in JSON or Markdown.
* `POST .../comparisons/:comparisonId/reviews` — Record review approval or changes-requested decision.

---

## 3. Internal Analysis Engine Contract (Python Analyzer)

The Python Analyzer exposes a stateless HTTP microservice operating internally at `http://analyzer:8000/api/v1`.

### 3.1 Health Check
* **`GET /api/v1/health`**
* **Response `200 OK`**:
  ```json
  {
    "status": "healthy",
    "version": "1.0.0",
    "engine": "tsql-ast-engine-v1"
  }
  ```

### 3.2 SQL Parse & Object Extraction
* **`POST /api/v1/parse`**
* **Request Payload**:
  ```json
  {
    "snapshotId": "snap-12345",
    "files": [
      {
        "relativePath": "tables/dbo.Customers.sql",
        "content": "CREATE TABLE dbo.Customers (CustomerId INT PRIMARY KEY, Email VARCHAR(100) NOT NULL);"
      },
      {
        "relativePath": "views/dbo.ActiveCustomers.sql",
        "content": "CREATE VIEW dbo.ActiveCustomers AS SELECT CustomerId, Email FROM dbo.Customers;"
      }
    ]
  }
  ```
* **Response Payload `200 OK`**:
  ```json
  {
    "snapshotId": "snap-12345",
    "status": "SUCCESS",
    "objects": [
      {
        "id": "dbo.Customers",
        "schema": "dbo",
        "name": "Customers",
        "type": "TABLE",
        "sourceFile": "tables/dbo.Customers.sql",
        "location": { "lineStart": 1, "lineEnd": 1, "colStart": 1, "colEnd": 81 },
        "columns": [
          { "name": "CustomerId", "dataType": "INT", "isNullable": false, "isPrimaryKey": true },
          { "name": "Email", "dataType": "VARCHAR", "length": 100, "isNullable": false }
        ],
        "constraints": [
          { "name": "PK_Customers", "type": "PRIMARY_KEY", "columns": ["CustomerId"] }
        ]
      },
      {
        "id": "dbo.ActiveCustomers",
        "schema": "dbo",
        "name": "ActiveCustomers",
        "type": "VIEW",
        "sourceFile": "views/dbo.ActiveCustomers.sql",
        "location": { "lineStart": 1, "lineEnd": 1, "colStart": 1, "colEnd": 79 }
      }
    ],
    "dependencies": [
      {
        "source": "dbo.ActiveCustomers",
        "target": "dbo.Customers",
        "type": "QUERIES_VIEW_OR_TABLE",
        "columnsReferenced": ["CustomerId", "Email"],
        "isConfirmed": true,
        "confidence": 1.0
      }
    ],
    "syntaxWarnings": []
  }
  ```

### 3.3 Schema Comparison & Impact Analysis
* **`POST /api/v1/compare`**
* **Request Payload**:
  Passes the parsed snapshot object structures (`base` and `target`) as extracted in the parse phase.
  ```json
  {
    "comparisonId": "comp-789",
    "base": {
      "snapshotId": "snap-1",
      "objects": [ ... ],
      "dependencies": [ ... ]
    },
    "target": {
      "snapshotId": "snap-2",
      "objects": [ ... ],
      "dependencies": [ ... ]
    }
  }
  ```
* **Response Payload `200 OK`**:
  ```json
  {
    "comparisonId": "comp-789",
    "analyzerVersion": "1.0.0",
    "ruleVersion": "1.0.0",
    "summary": {
      "objectsAdded": 0,
      "objectsRemoved": 0,
      "objectsModified": 1,
      "criticalFindings": 1,
      "highFindings": 0,
      "mediumFindings": 0,
      "lowFindings": 0
    },
    "deltas": [
      {
        "objectId": "dbo.Customers",
        "changeType": "MODIFIED",
        "columnChanges": [
          {
            "name": "Email",
            "changeType": "DROPPED"
          }
        ]
      }
    ],
    "findings": [
      {
        "ruleId": "RULE_DROPPED_REFERENCED_COLUMN",
        "severity": "CRITICAL",
        "title": "Column 'Email' dropped while referenced by view 'dbo.ActiveCustomers'",
        "description": "Table 'dbo.Customers' had column 'Email' removed, but view 'dbo.ActiveCustomers' depends directly on this column.",
        "changedObjectId": "dbo.Customers",
        "affectedObjectId": "dbo.ActiveCustomers",
        "dependencyPath": [
          { "from": "dbo.ActiveCustomers", "to": "dbo.Customers", "relation": "QUERIES_VIEW_OR_TABLE" }
        ],
        "confidence": "HIGH",
        "evidence": {
          "file": "tables/dbo.Customers.sql",
          "line": 1,
          "snippet": "CREATE TABLE dbo.Customers (CustomerId INT PRIMARY KEY);"
        },
        "verificationSteps": [
          "Verify if dbo.ActiveCustomers view can be rewritten without 'Email'.",
          "If column deprecation is intended, update or deprecate dependent views first."
        ]
      }
    ]
  }
  ```

---

## 4. Error Handling and Status Codes

| HTTP Status | Meaning & Context |
| :--- | :--- |
| `200 OK` | Successful request retrieval or synchronous analysis completion. |
| `201 Created` | Successfully registered user, created project, or uploaded snapshot. |
| `400 Bad Request` | Invalid payload schema, unsupported SQL upload format, or missing parameters. |
| `401 Unauthorized` | Missing or expired JWT credentials. |
| `403 Forbidden` | Insufficient workspace permissions (e.g., Reviewer attempting to delete project). |
| `404 Not Found` | Entity not found within the specified workspace. |
| `422 Unprocessable` | Semantic validation failure (e.g., base and target snapshots are identical). |
| `500 Server Error` | Unexpected backend or analyzer failure with structured incident identifier. |
