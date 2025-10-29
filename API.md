# API Documentation

This document describes the API endpoints for InMyControl. The API consists of both direct Supabase client interactions and serverless endpoints hosted on Vercel.

## Authentication

All API requests (except public endpoints) require authentication via Supabase Auth JWT tokens.

### Supabase Client Authentication

The frontend uses the Supabase client library for most CRUD operations. Authentication is handled automatically through the client:

```typescript
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)

// Get current session
const { data: { session } } = await supabase.auth.getSession()
```

### Serverless Endpoint Authentication

Serverless endpoints on Vercel must verify the Supabase JWT:

```typescript
import { createClient } from '@supabase/supabase-js'

export default async function handler(req: Request, res: Response) {
  const authHeader = req.headers.authorization
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Unauthorized' })
  }

  const token = authHeader.substring(7)
  const supabase = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )

  const { data: { user }, error } = await supabase.auth.getUser(token)
  if (error || !user) {
    return res.status(401).json({ error: 'Unauthorized' })
  }

  // Proceed with authenticated request
}
```

## Endpoint Reference

### Dashboard

#### GET /api/dashboard

Get aggregated dashboard data for the authenticated user.

**Authentication:** Required

**Response:**
```json
{
  "daily": {
    "recurring_tasks": [
      {
        "id": "uuid",
        "title": "Morning Exercise",
        "category_id": "PH",
        "completed_today": true,
        "target_count": null
      }
    ],
    "one_time_tasks": [
      {
        "id": "uuid",
        "title": "Doctor appointment",
        "due_date": "2024-12-15",
        "completed": false,
        "calendar_added": true
      }
    ]
  },
  "weekly": {
    "one_time_tasks": [...],
    "recurring_tasks": [...],
    "progress": {
      "completed": 5,
      "target": 10
    }
  },
  "monthly": {
    "progress": {
      "completed": 20,
      "target": 40
    }
  },
  "big_goal": {
    "id": "uuid",
    "title": "Run a marathon",
    "description": "...",
    "is_active": true
  }
}
```

**Implementation:** Can be implemented as a serverless function or as aggregated queries from the frontend using Supabase client.

### Recurring Tasks

#### GET /api/recurring_tasks

List all recurring tasks for the authenticated user.

**Authentication:** Required

**Query Parameters:**
- `type_id` (optional): Filter by task type (D, W, M)
- `active` (optional): Filter by active status (true/false)
- `category_id` (optional): Filter by category

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "owner_id": "uuid",
      "type_id": "D",
      "category_id": "PH",
      "title": "Morning Exercise",
      "description": "30 minutes of cardio",
      "repeat_rule": {"days": ["Mon", "Wed", "Fri"]},
      "target_count": null,
      "active": true,
      "created_at": "2024-01-01T00:00:00Z",
      "last_modified": "2024-01-01T00:00:00Z"
    }
  ]
}
```

**Alternative:** Use Supabase client directly:
```typescript
const { data, error } = await supabase
  .from('recurring_tasks')
  .select('*')
  .eq('owner_id', userId)
```

#### POST /api/recurring_tasks

Create a new recurring task.

**Authentication:** Required

**Request Body:**
```json
{
  "type_id": "D",
  "category_id": "PH",
  "title": "Morning Exercise",
  "description": "30 minutes of cardio",
  "repeat_rule": {"days": ["Mon", "Wed", "Fri"]},
  "target_count": null,
  "active": true
}
```

**Response:**
```json
{
  "id": "uuid",
  "owner_id": "uuid",
  ...
}
```

#### PUT /api/recurring_tasks/:id

Update an existing recurring task.

**Authentication:** Required

**Request Body:** Same as POST (all fields optional)

**Response:** Updated task object

#### DELETE /api/recurring_tasks/:id

Delete a recurring task and associated logs.

**Authentication:** Required

**Response:**
```json
{
  "success": true
}
```

### One-time Tasks

#### GET /api/one_time_tasks

List all one-time tasks for the authenticated user.

**Authentication:** Required

**Query Parameters:**
- `due_date_start` (optional): Filter tasks with due_date >= this date
- `due_date_end` (optional): Filter tasks with due_date <= this date
- `completed` (optional): Filter by completion status
- `category_id` (optional): Filter by category

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "owner_id": "uuid",
      "category_id": "MH",
      "title": "Doctor appointment",
      "description": "Annual checkup",
      "due_date": "2024-12-15",
      "calendar_added": true,
      "calendar_event_id": "google_event_id",
      "completed": false,
      "created_at": "2024-01-01T00:00:00Z",
      "last_modified": "2024-01-01T00:00:00Z"
    }
  ]
}
```

#### POST /api/one_time_tasks

Create a new one-time task.

**Authentication:** Required

**Request Body:**
```json
{
  "category_id": "MH",
  "title": "Doctor appointment",
  "description": "Annual checkup",
  "due_date": "2024-12-15"
}
```

**Response:** Created task object

#### PUT /api/one_time_tasks/:id

Update an existing one-time task.

**Authentication:** Required

**Request Body:** Same as POST (all fields optional)

**Response:** Updated task object

#### DELETE /api/one_time_tasks/:id

Delete a one-time task.

**Authentication:** Required

**Response:**
```json
{
  "success": true
}
```

#### POST /api/one_time_tasks/:id/add_to_google

Add a one-time task to Google Calendar.

**Authentication:** Required

**Request Body:**
```json
{
  "time": "14:00", // optional, defaults to all-day
  "duration_minutes": 60 // optional, defaults to 60
}
```

**Response:**
```json
{
  "success": true,
  "calendar_event_id": "google_event_id",
  "calendar_url": "https://calendar.google.com/calendar/event?eid=..."
}
```

**Errors:**
- `401`: Not authenticated
- `403`: Google OAuth not configured or token missing
- `404`: Task not found
- `500`: Google Calendar API error

**Implementation Notes:**
- This endpoint requires Google OAuth token
- Can be implemented client-side using Google Calendar API directly
- If server-side, requires storing encrypted refresh token or requesting user permission each time

### Internal Calendar

#### GET /api/calendar/internal

Get internal calendar events for a date range.

**Authentication:** Required

**Query Parameters:**
- `start` (required): Start date (YYYY-MM-DD)
- `end` (required): End date (YYYY-MM-DD)

**Response:**
```json
{
  "events": [
    {
      "date": "2024-12-15",
      "event_type": "recurring_completion",
      "event_data": {
        "task_id": "uuid",
        "task_title": "Morning Exercise",
        "category_id": "PH",
        "completed": true,
        "count": 1
      }
    },
    {
      "date": "2024-12-15",
      "event_type": "one_time_task",
      "event_data": {
        "task_id": "uuid",
        "task_title": "Doctor appointment",
        "category_id": "MH",
        "completed": false,
        "calendar_added": true
      }
    }
  ]
}
```

**Alternative:** Use database function:
```typescript
const { data, error } = await supabase.rpc('get_internal_calendar_events', {
  p_owner_id: userId,
  p_start_date: startDate,
  p_end_date: endDate
})
```

### Task Logs

#### POST /api/task_logs

Create or update a task log entry.

**Authentication:** Required

**Request Body:**
```json
{
  "task_id": "uuid",
  "date": "2024-12-15",
  "completed": true,
  "count": 1,
  "note": "Felt great!"
}
```

**Response:** Created/updated log entry

**Alternative:** Use Supabase client with upsert:
```typescript
const { data, error } = await supabase
  .from('task_logs')
  .upsert({
    task_id: taskId,
    owner_id: userId,
    date: date,
    completed: true,
    count: 1,
    note: "Felt great!"
  }, {
    onConflict: 'task_id,date'
  })
```

#### GET /api/task_logs

Get task logs for the authenticated user.

**Authentication:** Required

**Query Parameters:**
- `date` (optional): Filter by specific date
- `task_id` (optional): Filter by task
- `start_date` (optional): Filter logs >= this date
- `end_date` (optional): Filter logs <= this date

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "task_id": "uuid",
      "owner_id": "uuid",
      "date": "2024-12-15",
      "completed": true,
      "count": 1,
      "note": "Felt great!",
      "created_at": "2024-12-15T08:00:00Z"
    }
  ]
}
```

### Templates

#### GET /api/templates

List available templates.

**Authentication:** Not required (templates are public)

**Response:**
```json
{
  "data": [
    {
      "id": "uuid",
      "owner_id": "uuid",
      "name": "Fitness Starter",
      "description": "Basic fitness routine",
      "seed_payload": {...},
      "created_at": "2024-01-01T00:00:00Z"
    }
  ]
}
```

#### POST /api/templates/:id/apply

Apply a template to the authenticated user's account.

**Authentication:** Required

**Request Body:**
```json
{
  "overwrite_existing": false // optional, defaults to false
}
```

**Response:**
```json
{
  "success": true,
  "created": {
    "categories": 2,
    "recurring_tasks": 5,
    "one_time_tasks": 3
  }
}
```

**Implementation:**
- Creates categories from seed_payload (if owner_id is NULL or overwrite_existing is true)
- Creates recurring_tasks with owner_id = current user
- Creates one_time_tasks with owner_id = current user
- Returns count of created items

### Quotes

#### GET /api/quotes/random

Get a random quote (global or user-specific).

**Authentication:** Not required (but can filter user quotes if authenticated)

**Query Parameters:**
- `include_user` (optional): Include user-specific quotes if authenticated (default: true)

**Response:**
```json
{
  "id": "uuid",
  "owner_id": null,
  "quote": "The only way to do great work is to love what you do.",
  "reference": "Steve Jobs",
  "created_at": "2024-01-01T00:00:00Z"
}
```

**Implementation:** Random selection from global quotes (owner_id IS NULL) and optionally user quotes.

### Export / Import

#### GET /api/export

Export all user data as JSON.

**Authentication:** Required

**Response:**
```json
{
  "user": {
    "email": "user@example.com",
    "display_name": "John Doe",
    "settings": {...}
  },
  "big_goals": [...],
  "categories": [...],
  "recurring_tasks": [...],
  "one_time_tasks": [...],
  "task_logs": [...],
  "quotes": [...],
  "exported_at": "2024-12-15T12:00:00Z"
}
```

**Implementation:**
- Queries all tables for owner_id = current user
- Includes all user-owned data
- Returns JSON download or JSON response

#### POST /api/import

Import user data from JSON export.

**Authentication:** Required

**Request Body:**
```json
{
  "data": {
    "big_goals": [...],
    "categories": [...],
    "recurring_tasks": [...],
    "one_time_tasks": [...],
    "task_logs": [...],
    "quotes": [...]
  },
  "merge_strategy": "overwrite" // or "skip_existing"
}
```

**Response:**
```json
{
  "success": true,
  "imported": {
    "big_goals": 1,
    "categories": 3,
    "recurring_tasks": 10,
    "one_time_tasks": 5,
    "task_logs": 50,
    "quotes": 2
  }
}
```

**Validation:**
- Validate JSON structure
- Ensure all foreign key references are valid
- Set owner_id to current user for all imported items
- Handle conflicts based on merge_strategy

## Error Responses

All endpoints may return the following error responses:

```json
{
  "error": "Error message",
  "code": "ERROR_CODE",
  "details": {} // optional additional details
}
```

**Common HTTP Status Codes:**
- `200`: Success
- `201`: Created
- `400`: Bad Request (validation error)
- `401`: Unauthorized (not authenticated)
- `403`: Forbidden (not authorized)
- `404`: Not Found
- `409`: Conflict (e.g., duplicate entry)
- `500`: Internal Server Error

## Rate Limiting

Consider implementing rate limiting for serverless endpoints to prevent abuse:

- Per-user rate limits (e.g., 100 requests/minute)
- Global rate limits for unauthenticated endpoints
- Monitor via Vercel analytics and Supabase logs

## Implementation Strategy

### Client-side (Supabase Direct)

Use Supabase client for most CRUD operations:
- Recurring tasks CRUD
- One-time tasks CRUD
- Task logs CRUD外套
- Categories CRUD
- Quotes retrieval
- User profile updates

### Serverless Endpoints (Vercel)

Use serverless functions for:
- Google Calendar integration (POST /api/one_time_tasks/:id/add_to_google)
- Template application (POST /api/templates/:id/apply)
- Data export (GET /api/export)
- Data import (POST /api/import)
- Aggregated dashboard data (if complex queries needed)

This approach minimizes serverless function usage while keeping sensitive operations (like Google OAuth) secure.

