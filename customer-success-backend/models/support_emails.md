# support_emails

Managed mailboxes that are actively monitored via Gmail API watch().

## Schema

| Column | Type | Default | Description |
|--------|------|---------|-------------|
| id | UUID | `gen_random_uuid()` | PK |
| email_address | TEXT | — | Gmail/Workspace address (UNIQUE) |
| display_name | TEXT | NULL | Friendly name |
| is_active | BOOLEAN | true | Toggle monitoring on/off |
| watch_expiration | TIMESTAMPTZ | NULL | When Gmail watch() expires (~7 days) |
| watch_history_id | TEXT | NULL | Last known historyId for incremental sync |
| created_at | TIMESTAMPTZ | `now()` | |
| updated_at | TIMESTAMPTZ | `now()` | |

## Indexes

| Name | Columns | Type |
|------|---------|------|
| idx_support_emails_active | is_active | Partial (WHERE is_active = true) |

## Relationships

- **emails** → `support_email_id` FK (CASCADE delete)
