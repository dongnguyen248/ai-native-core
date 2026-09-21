# Technical Design: Work Order Domain (WO-201)

## 1. Overview

This document defines the work order creation flow for Issue **WO-201**, including user-interface validation, the PostgreSQL persistence model, and the REST API contract.

The work order is created in `DRAFT` status. A work order must reference an active customer, and its priority defaults to `MED` when the caller does not provide a value.

## 2. UI Validation Matrix

| Field Name | Type | Required | Default | Rules / Constraints |
| :--- | :--- | :--- | :--- | :--- |
| `title` | String | Yes | None | Minimum 5 characters; maximum 255 characters |
| `description` | String | No | None | Maximum 2000 characters |
| `priority` | Enum | Yes | `MED` | Allowed values: `LOW`, `MED`, `HIGH`, `CRITICAL` |
| `customer_id` | UUID | Yes | None | Must reference a valid active Customer |

### 2.1 Validation Behavior

- The UI must prevent submission when `title` is missing or outside the 5-255 character range.
- The UI must prevent submission when `description` exceeds 2000 characters.
- The priority control must only allow `LOW`, `MED`, `HIGH`, or `CRITICAL`; `MED` is selected by default.
- The customer selector must return an active Customer and submit its UUID as `customer_id`.
- Server-side validation remains authoritative and must repeat these constraints for every request.

## 3. PostgreSQL Data Schema

The `work_orders` table stores the work order identity, customer association, workflow state, priority, and creation timestamp.

```sql
CREATE TABLE work_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    priority VARCHAR(20) NOT NULL DEFAULT 'MED',
    status VARCHAR(20) NOT NULL DEFAULT 'DRAFT',
    customer_id UUID NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_priority CHECK (priority IN ('LOW', 'MED', 'HIGH', 'CRITICAL'))
);
```

`customer_id` must be validated against the Customer domain to ensure that the referenced customer exists and is active before insertion. The schema requires the `pgcrypto` extension, or an equivalent UUID generation capability, for `gen_random_uuid()`.

## 4. REST API Contract

### 4.1 Create Work Order

**Method:** `POST`
**Path:** `/api/v1/work-orders`
**Authentication:** `Authorization: Bearer <token>`

#### Request Headers

```http
Authorization: Bearer <token>
Content-Type: application/json
```

#### Request Body

```json
{
  "title": "HVAC Repair Unit 4",
  "description": "System reporting error code E-42",
  "priority": "HIGH",
  "customer_id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11"
}
```

#### Request Contract

| Property | Type | Required | Constraints |
| :--- | :--- | :--- | :--- |
| `title` | String | Yes | 5-255 characters |
| `description` | String | No | At most 2000 characters |
| `priority` | String enum | No | `LOW`, `MED`, `HIGH`, `CRITICAL`; defaults to `MED` |
| `customer_id` | UUID | Yes | Must identify an active Customer |

#### Successful Response: `201 Created`

```json
{
  "id": "c9bf9e57-1685-4c89-bafb-ff5af830be8a",
  "status": "DRAFT",
  "created_at": "2026-08-30T10:00:00Z"
}
```

#### Response Contract

| Property | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | Unique identifier assigned to the new work order |
| `status` | String | Initial workflow status; always `DRAFT` on creation |
| `created_at` | ISO 8601 timestamp | UTC creation timestamp |

The API must reject unauthenticated requests and requests that fail the validation rules above. Validation and authorization error response shapes are outside the scope of this WO-201 contract.
