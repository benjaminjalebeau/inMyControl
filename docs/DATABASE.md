# Database Schema and Migration Guide

This document provides detailed SQL schema definitions for the InMyControl database, including table structures, indexes, constraints, and Row-Level Security (RLS) policies.

## Prerequisites

- Supabase project created
- Database access configured
- Migration tool or SQL editor ready

## Table Definitions

### users

Extended user profile table linked to Supabase Auth users.

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    email TEXT NOT NULL,
    display_name TEXT,
    profile_image_url TEXT,
    settings JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at);

-- Trigger to update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### big_goals

User-defined motivating goals.

```sql
CREATE TABLE big_goals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    description TEXT,
    image_url TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_modified TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_big_goals_owner_id ON big_goals(owner_id);
CREATE INDEX idx_big_goals_is_active ON big_goals(is_active, owner_id);

CREATE TRIGGER update_big_goals_last_modified BEFORE UPDATE ON big_goals
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### task_types

Reference table for task type classifications (D=Daily, W=Weekly, M=Monthly, O=One-time).

```sql
CREATE TABLE task_types (
    id TEXT PRIMARY KEY,
    description TEXT NOT NULL,
    icon TEXT
);

-- Seed data
INSERT INTO task_types (id, description, icon) VALUES
    ('D', 'Daily', 'calendar-day'),
    ('W', 'Weekly', 'calendar-week'),
    ('M', 'Monthly', 'calendar-month'),
    ('O', 'One-time', 'calendar-event');
```

### categories

Task categories (global or user-specific).

```sql
CREATE TABLE categories (
    id TEXT PRIMARY KEY,
    description TEXT NOT NULL,
    owner_id UUID REFERENCES users(id) ON DELETE CASCADE,
    color TEXT, -- hex color code
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_categories_owner_id ON categories(owner_id);

-- Example global categories
INSERT INTO categories (id, description, owner_id, color) VALUES
    ('PH', 'Physical Health', NULL, '#10B981'),
    ('MH', 'Mental Health', NULL, '#3B82F6'),
    ('SH', 'Social Health', NULL, '#8B5CF6'),
    ('RH', 'Relational Health', NULL, '#F59E0B');
```

### recurring_tasks

Recurring tasks with flexible repeat rules stored as JSONB.

```sql
CREATE TABLE recurring_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type_id TEXT NOT NULL REFERENCES task_types(id),
    category_id TEXT NOT NULL REFERENCES categories(id),
    title TEXT NOT NULL,
    description TEXT,
    repeat_rule JSONB NOT NULL DEFAULT '{}'::jsonb,
    target_count INTEGER,
    active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_modified TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_recurring_tasks_owner_id ON recurring_tasks(owner_id);
CREATE INDEX idx_recurring_tasks_type_id ON recurring_tasks(type_id);
CREATE INDEX idx_recurring_tasks_category_id ON recurring_tasks(category_id);
CREATE INDEX idx_recurring_tasks_active ON recurring_tasks(owner_id, active);
CREATE INDEX idx_recurring_tasks_repeat_rule ON recurring_tasks USING GIN (repeat_rule);

CREATE TRIGGER update_recurring_tasks_last_modified BEFORE UPDATE ON recurring_tasks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

**repeat_rule JSONB examples:**
```json
// Daily: specific weekdays
{"days": ["Mon", "Wed", "Fri"]}

// Weekly: complete once per week
{"weekly": true}

// Weekly: target count (e.g., 3x per week)
{"weekly_target": 3}

// Monthly: complete once per month
{"monthly": true}

// Monthly: target count (e.g., 15x per month)
{"monthly_target": 15}
```

### one_time_tasks

One-time tasks with optional Google Calendar integration.

```sql
CREATE TABLE one_time_tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id TEXT NOT NULL REFERENCES categories(id),
    title TEXT NOT NULL,
    description TEXT,
    due_date DATE,
    calendar_added BOOLEAN DEFAULT false,
    calendar_event_id TEXT,
    completed BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_modified TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_one_time_tasks_owner_id ON one_time_tasks(owner_id);
CREATE INDEX idx_one_time_tasks_category_id ON one_time_tasks(category_id);
CREATE INDEX idx_one_time_tasks_due_date ON one_time_tasks(owner_id, due_date);
CREATE INDEX idx_one_time_tasks_completed ON one_time_tasks(owner_id, completed);
CREATE INDEX idx_one_time_tasks_calendar_added ON one_time_tasks(calendar_added) WHERE calendar_added = true;

CREATE TRIGGER update_one_time_tasks_last_modified BEFORE UPDATE ON one_time_tasks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### task_logs

Completion logs for recurring tasks.

```sql
CREATE TABLE task_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id UUID NOT NULL REFERENCES recurring_tasks(id) ON DELETE CASCADE,
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    completed BOOLEAN DEFAULT true,
    count INTEGER DEFAULT 1,
    note TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(task_id, date)
);

CREATE INDEX idx_task_logs_task_id ON task_logs(task_id);
CREATE INDEX idx_task_logs_owner_id ON task_logs(owner_id);
CREATE INDEX idx_task_logs_date ON task_logs(owner_id, date);
CREATE INDEX idx_task_logs_completed ON task_logs(task_id, date, completed);
```

### quotes

Motivational quotes (global or user-specific).

```sql
CREATE TABLE quotes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES users(id) ON DELETE CASCADE,
    quote TEXT NOT NULL,
    reference TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_quotes_owner_id ON quotes(owner_id);
CREATE INDEX idx_quotes_global ON quotes(owner_id) WHERE owner_id IS NULL;
```

### attachments

File attachments for profiles and tasks.

```sql
CREATE TABLE attachments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    parent_type TEXT NOT NULL, -- 'user', 'recurring_task', 'one_time_task'
    parent_id UUID NOT NULL,
    storage_path TEXT NOT NULL,
    public BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_attachments_owner_id ON attachments(owner_id);
CREATE INDEX idx_attachments_parent ON attachments(parent_type, parent_id);
CREATE INDEX idx_attachments_public ON attachments(public) WHERE public = true;
```

### templates

Template definitions for seeding user content.

```sql
CREATE TABLE templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID REFERENCES users(id) ON DELETE SET NULL,
    name TEXT NOT NULL,
    description TEXT,
    seed_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_templates_owner_id ON templates(owner_id);
CREATE INDEX idx_templates_seed_payload ON templates USING GIN (seed_payload);
```

**seed_payload JSONB example:**
```json
{
  "categories": [
    {"id": "CUSTOM1", "description": "Custom Category", "color": "#FF5733"}
  ],
  "recurring_tasks": [
    {
      "type_id": "D",
      "category_id": "PH",
      "title": "Morning Exercise",
      "repeat_rule": {"days": ["Mon", "Tue", "Wed", "Thu", "Fri"]}
    }
  ],
  "one_time_tasks": [
    {
      "category_id": "MH",
      "title": "Schedule therapy session",
      "due_date": "2024-12-31"
    }
  ]
}
```

### audit_logs

Audit trail for user actions (optional but recommended).

```sql
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    event_type TEXT NOT NULL,
    payload JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_owner_id ON audit_logs(owner_id);
CREATE INDEX idx_audit_logs_event_type ON audit_logs(event_type);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
CREATE INDEX idx_audit_logs_payload ON audit_logs USING GIN (payload);
```

## Row-Level Security (RLS) Policies

Enable RLS on all tables and define policies:

```sql
-- Enable RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE big_goals ENABLE ROW LEVEL SECURITY;
ALTER TABLE categories ENABLE ROW LEVEL SECURITY;
ALTER TABLE recurring_tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE one_time_tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE task_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE quotes ENABLE ROW LEVEL SECURITY;
ALTER TABLE attachments ENABLE ROW LEVEL SECURITY;
ALTER TABLE templates ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

-- Users: can read/update own profile
CREATE POLICY "Users can view own profile" ON users
    FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Users can update own profile" ON users
    FOR UPDATE USING (auth.uid() = id);

CREATE POLICY "Users can insert own profile" ON users
    FOR INSERT WITH CHECK (auth.uid() = id);

-- Big Goals: owner access only
CREATE POLICY "Users can manage own big goals" ON big_goals
    FOR ALL USING (auth.uid() = owner_id);

-- Categories: global categories readable by all, user categories by owner
CREATE POLICY "Anyone can view global categories" ON categories
    FOR SELECT USING (owner_id IS NULL);

CREATE POLICY "Users can view own categories" ON categories
    FOR SELECT USING (auth.uid() = owner_id);

CREATE POLICY "Users can manage own categories" ON categories
    FOR ALL USING (auth.uid() = owner_id);

-- Recurring Tasks: owner access only
CREATE POLICY "Users can manage own recurring tasks" ON recurring_tasks
    FOR ALL USING (auth.uid() = owner_id);

-- One-time Tasks: owner access only
CREATE POLICY "Users can manage own one-time tasks" ON one_time_tasks
    FOR ALL USING (auth.uid() = owner_id);

-- Task Logs: owner access only
CREATE POLICY "Users can manage own task logs" ON task_logs
    FOR ALL USING (auth.uid() = owner_id);

-- Quotes: global quotes readable by all, user quotes by owner
CREATE POLICY "Anyone can view global quotes" ON quotes
    FOR SELECT USING (owner_id IS NULL);

CREATE POLICY "Users can view own quotes" ON quotes
    FOR SELECT USING (auth.uid() = owner_id);

CREATE POLICY "Users can manage own quotes" ON quotes
    FOR ALL USING (auth.uid() = owner_id);

-- Attachments: owner access only
CREATE POLICY "Users can manage own attachments" ON attachments
    FOR ALL USING (auth.uid() = owner_id);

-- Templates: readable by all, manageable by owner (admin)
CREATE POLICY "Anyone can view templates" ON templates
    FOR SELECT USING (true);

CREATE POLICY "Users can manage own templates" ON templates
    FOR ALL USING (auth.uid() = owner_id);

-- Audit Logs: owner read-only
CREATE POLICY "Users can view own audit logs" ON audit_logs
    FOR SELECT USING (auth.uid() = owner_id);

CREATE POLICY "System can insert audit logs" ON audit_logs
    FOR INSERT WITH CHECK (auth.uid() = owner_id);
```

## Storage Buckets

Configure Supabase Storage buckets:

```sql
-- Profile images bucket
INSERT INTO storage.buckets (id, name, public)
VALUES ('profile-images', 'profile-images', false);

-- Attachments bucket
INSERT INTO storage.buckets (id, name, public)
VALUES ('attachments', 'attachments', false);

-- Storage policies for profile images
CREATE POLICY "Users can upload own profile images" ON storage.objects
    FOR INSERT WITH CHECK (
        bucket_id = 'profile-images' AND
        auth.uid()::text = (storage.foldername(name))[1]
    );

CREATE POLICY "Users can view own profile images" ON storage.objects
    FOR SELECT USING (
        bucket_id = 'profile-images' AND
        auth.uid()::text = (storage.foldername(name))[1]
    );

CREATE POLICY "Users can update own profile images" ON storage.objects
    FOR UPDATE USING (
        bucket_id = 'profile-images' AND
        auth.uid()::text = (storage.foldername(name))[1]
    );

CREATE POLICY "Users can delete own profile images" ON storage.objects
    FOR DELETE USING (
        bucket_id = 'profile-images' AND
        auth.uid()::text = (storage.foldername(name))[1]
    );

-- Storage policies for attachments
CREATE POLICY "Users can upload own attachments" ON storage.objects
    FOR INSERT WITH CHECK (
        bucket_id = 'attachments' AND
        auth.uid()::text = (storage.foldername(name))[1]
    );

CREATE POLICY "Users can view own attachments" ON storage.objects
    FOR SELECT USING (
        bucket_id = 'attachments' AND
        auth.uid()::text = (storage.foldername(name))[1]
    );

CREATE POLICY "Users can update own attachments" ON storage.objects
    FOR UPDATE USING (
        bucket_id = 'attachments' AND
        auth.uid()::text = (storage.foldername(name))[1]
    );

CREATE POLICY "Users can delete own attachments" ON storage.objects
    FOR DELETE USING (
        bucket_id = 'attachments' AND
        auth.uid()::text = (storage.foldername(name))[1]
    );
```

## Functions and Triggers

### Helper function for updated_at timestamps

Already defined above; reuse for all tables with `last_modified` or `updated_at` columns.

### Function to derive calendar events (optional)

```sql
CREATE OR REPLACE FUNCTION get_internal_calendar_events(
    p_owner_id UUID,
    p_start_date DATE,
    p_end_date DATE
)
RETURNS TABLE (
    date DATE,
    event_type TEXT,
    event_data JSONB
) AS $$
BEGIN
    RETURN QUERY
    -- Recurring task completions
    SELECT
        tl.date,
        'recurring_completion'::TEXT,
        jsonb_build_object(
            'task_id', rt.id,
            'task_title', rt.title,
            'category_id', rt.category_id,
            'completed', tl.completed,
            'count', tl.count
        ) AS event_data
    FROM task_logs tl
    JOIN recurring_tasks rt ON tl.task_id = rt.id
    WHERE tl.owner_id = p_owner_id
        AND tl.date >= p_start_date
        AND tl.date <= p_end_date
        AND rt.active = true
    
    UNION ALL
    
    -- One-time tasks
    SELECT
        ott.due_date AS date,
        'one_time_task'::TEXT,
        jsonb_build_object(
            'task_id', ott.id,
            'task_title', ott.title,
            'category_id', ott.category_id,
            'completed', ott.completed,
            'calendar_added', ott.calendar_added,
            'calendar_event_id', ott.calendar_event_id
        ) AS event_data
    FROM one_time_tasks ott
    WHERE ott.owner_id = p_owner_id
        AND ott.due_date >= p_start_date
        AND ott.due_date <= p_end_date
        AND ott.due_date IS NOT NULL
    
    ORDER BY date;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Migration Notes

1. Run migrations in order:
   - Base tables (users, task_types, categories)
   - Feature tables (big_goals, recurring_tasks, one_time_tasks, task_logs)
   - Supporting tables (quotes, attachments, templates, audit_logs)
   - RLS policies
   - Storage buckets and policies
   - Helper functions

2. Seed initial data:
   - task_types (D, W, M, O)
   - Global categories (PH, MH, SH, RH)

3. Test RLS policies:
   - Create test users
   - Verify cross-user access is blocked
   - Verify global categories/quotes are readable by all

4. Monitor indexes:
   - Use Supabase dashboard to monitor query performance
   - Add additional indexes based on query patterns

