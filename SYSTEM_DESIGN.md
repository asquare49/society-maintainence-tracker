# System Design Write-Up: Society Maintenance Tracker

## 1. Complaint History Model
Maintaining a transparent, immutable record of every complaint state transition is vital for accountability between apartment residents and society administration.

### Data Model & Schema Design
The complaint lifecycle is modeled using a relational design between two core entities: `Complaint` and `ComplaintHistory`.
- `Complaint`: Holds the master state, including current `status` (`OPEN`, `IN_PROGRESS`, `RESOLVED`), `priority` (`LOW`, `MEDIUM`, `HIGH`), `category`, `photoUrl`, `userId`, `resolvedAt`, and timestamps.
- `ComplaintHistory`: An append-only audit ledger where every state transition or priority modification creates an unalterable history record containing:
  - `id`: Unique identifier
  - `complaintId`: Foreign key to `Complaint` (with cascade semantics)
  - `actorId` & `actorName`: Identifies whether the action was taken by the Resident or Admin
  - `actorRole`: `RESIDENT` or `ADMIN`
  - `fromStatus` & `toStatus`: Represents the exact state vector (e.g., `OPEN` $\to$ `IN_PROGRESS`)
  - `note`: Optional explanation or technician feedback (e.g., parts replaced, contractor scheduled)
  - `createdAt`: ISO-8601 audit timestamp

### State Transition Guarantees
All status updates execute within atomic database transactions (`prisma.$transaction`). When a complaint transitions to `RESOLVED`, `resolvedAt` is automatically set and the issue is considered closed. If a ticket is reopened, history logs preserve the entire prior resolution attempts and reasons.

---

## 2. Overdue Detection System
To ensure resident grievances do not languish unattended, the platform features a dynamic, configurable overdue detection engine.

### Configurable Thresholds & Evaluation Strategy
- The overdue limit is stored in a `SystemSetting` table (`overdue_threshold_days`, default 3 days), customizable by administrators at runtime via the UI without requiring system redeployments.
- Overdue status is evaluated using an on-demand calculation pipeline:
  $$\text{Days Open} = \left\lfloor \frac{\text{Current Time} - \text{Created Time}}{24 \times 60 \times 60 \times 1000} \right\rfloor$$
  $$\text{Is Overdue} = (\text{Status} \neq \text{RESOLVED}) \land (\text{Days Open} \ge \text{Threshold Days})$$

### Priority Queue Surfacing
In the administration query interface, complaints are sorted using a multi-factor ranking algorithm:
1. **Overdue Flag**: Overdue complaints are surfaced to the very top with distinctive visual alert ribbons and counter badges.
2. **Lifecycle State**: Active tickets (`OPEN` / `IN_PROGRESS`) precede `RESOLVED` items.
3. **Priority Weight**: `HIGH` $\to$ `MEDIUM` $\to$ `LOW`.
4. **Chronological Recency**: Most recent timestamps.

---

## 3. Photo Handling & Storage
Supporting photo attachments helps maintenance technicians diagnose structural, plumbing, and electrical issues before visiting flats.

### Processing Pipeline
1. **Client Submission**: Residents capture or select photos on mobile or desktop and submit via multipart form data (`multipart/form-data`).
2. **Validation & Filtering**: The backend `Multer` middleware validates MIME types (`image/jpeg`, `image/png`, `image/webp`) and enforces a 5MB size limit.
3. **Storage Strategy**: Files are stored with cryptographically collision-resistant names (`complaint-<timestamp>-<random>.<ext>`) in a dedicated `uploads/` volume served statically via Express.
4. **Cloud-Ready Abstraction**: The file URL (`/uploads/...`) stored on the complaint record is an abstraction that seamlessly maps to local persistent volumes in self-hosted setups or cloud object stores (e.g., AWS S3, Cloudinary) in distributed deployments.

---

## 4. Notification Flow
Residents and administrators must stay synchronously and asynchronously aligned throughout the maintenance lifecycle.

```
+------------------+         +----------------------+         +-----------------------+
|  Resident/Admin  | ------> | Express REST Backend | ------> | Database & Transaction|
|    UI Action     |         +----------------------+         +-----------------------+
+------------------+                    |
                                        v
                            +-----------------------+
                            |     Email Service     |
                            |  (Nodemailer Engine)  |
                            +-----------------------+
                               /                 \
                              v                   v
                     [Status Update]         [Important Notice]
                     To: Ticket Owner        To: All Residents (BCC)
```

### Event Triggers
1. **Complaint Status Modification**: When an administrator moves a complaint between `OPEN`, `IN_PROGRESS`, or `RESOLVED`, an asynchronous notification task formats an HTML email detailing the new status, updating administrator, timestamp, and technician notes.
2. **Important Notice Broadcast**: When an administrator publishes an urgent announcement marked `isImportant`, the platform fetches active resident email addresses and sends a broadcast alert with notice details.

### Reliability & Resilience
Email dispatch executes asynchronously in a non-blocking worker pattern, ensuring email transport latency or third-party outages never block the core HTTP API transaction response. In development, test inboxes are generated on the fly via Ethereal SMTP with shareable preview URLs logged in real time.
