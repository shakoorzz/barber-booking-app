# Fix for AM/PM Time Display Issue

## Problem Description
Time slots in the barber booking application are displaying with both AM and PM labels (e.g., "2:00 PM AM" or "2:15 PM AM").

## Root Cause
The issue is in the PostgreSQL function `get_available_slots()` in the Supabase SQL code. Specifically, this line:

```sql
select to_char(slot_bounds.slot_local, 'HH12:MI AM')
```

The format string `'HH12:MI AM'` causes PostgreSQL's `to_char()` function to:
1. Format the hour as 12-hour format (HH12)
2. Add the minutes (MI)
3. Add a literal space and "AM" string
4. Then add ANOTHER AM/PM indicator (the AM at the end)

This results in duplicate AM/PM indicators.

## Solution

### Option 1: Simple Fix (Recommended)
Replace the problematic line with:

```sql
select to_char(slot_bounds.slot_local, 'HH:MI AM')
```

This uses:
- `HH` - Hour in 12-hour format (automatically inferred with AM/PM)
- `MI` - Minutes
- `AM` - AM/PM indicator

### Option 2: Explicit 12-Hour Format with Fill Mode
```sql
select to_char(slot_bounds.slot_local, 'FMHH12:FMMI AM')
```

This uses:
- `FM` - Fill mode (suppresses leading zeros)
- `HH12` - Hour in 12-hour format
- `MI` - Minutes  
- `AM` - AM/PM indicator

### Option 3: Consistent Leading Zeros
```sql
select to_char(slot_bounds.slot_local, 'HH12:MI AM')
```
Wait - this is the WRONG format. Use:
```sql
select to_char(slot_bounds.slot_local, 'HH12:MI TM')
```
Actually, the cleanest is still Option 1.

## Complete Updated SQL Function

```sql
-- Fixed function to properly format time slots
CREATE OR REPLACE FUNCTION get_available_slots(selected_date text, service_duration_minutes int)
RETURNS TABLE (available_slot text)
LANGUAGE plpgsql
AS $$
DECLARE
    barber_work_start time;
    barber_work_end time;
    barber_lunch_start time;
    barber_lunch_end time;
    day_start timestamptz;
    day_end timestamptz;
    local_midnight timestamp;
    work_start_ts timestamp;
    work_end_ts timestamp;
    lunch_start_ts timestamp;
    lunch_end_ts timestamp;
    target_timezone text := 'America/New_York';
BEGIN
    PERFORM set_config('timezone', target_timezone, true);

    day_start := (selected_date::date)::timestamptz;
    day_end := ((selected_date::date + interval '1 day')::timestamptz);

    SELECT work_start_time, work_end_time, lunch_start_time, lunch_end_time
    INTO barber_work_start, barber_work_end, barber_lunch_start, barber_lunch_end
    FROM public.barber_config
    LIMIT 1;

    -- Anchor barber schedule boundaries to the selected day in the target timezone
    local_midnight := (day_start AT TIME ZONE target_timezone);
    work_start_ts := local_midnight + (barber_work_start - time '00:00');
    work_end_ts := local_midnight + (barber_work_end - time '00:00');
    lunch_start_ts := CASE
        WHEN barber_lunch_start IS NULL THEN NULL
        ELSE local_midnight + (barber_lunch_start - time '00:00')
    END;
    lunch_end_ts := CASE
        WHEN barber_lunch_end IS NULL THEN NULL
        ELSE local_midnight + (barber_lunch_end - time '00:00')
    END;

    RETURN QUERY
    WITH potential_slots AS (
        SELECT generate_series(
            day_start,
            day_end - (service_duration_minutes || ' minutes')::interval,
            '15 minutes'::interval
        ) AS slot_start
    ),
    existing_appointments AS (
        SELECT start_time, end_time
        FROM public.appointments
        WHERE start_time >= day_start AND start_time < day_end
    )
    -- FIXED: Changed format string from 'HH12:MI AM' to 'HH:MI AM'
    SELECT to_char(slot_bounds.slot_local, 'HH:MI AM')
    FROM potential_slots p
    CROSS JOIN LATERAL (
        SELECT
            (p.slot_start AT TIME ZONE target_timezone) AS slot_local,
            ((p.slot_start + (service_duration_minutes || ' minutes')::interval) AT TIME ZONE target_timezone) AS slot_local_end
    ) AS slot_bounds
    WHERE slot_bounds.slot_local >= work_start_ts
      AND slot_bounds.slot_local_end <= work_end_ts
      AND (
            lunch_start_ts IS NULL OR lunch_end_ts IS NULL
            OR slot_bounds.slot_local_end <= lunch_start_ts
            OR slot_bounds.slot_local >= lunch_end_ts
          )
      AND NOT EXISTS (
          SELECT 1
          FROM existing_appointments ea
          WHERE (p.slot_start, (p.slot_start + (service_duration_minutes || ' minutes')::interval))
                OVERLAPS (ea.start_time, ea.end_time)
      )
    ORDER BY slot_bounds.slot_local;
END;
$$;
```

## Implementation Steps

1. **Access your Supabase project dashboard**
2. **Navigate to SQL Editor**
3. **Run the updated function** (the complete SQL above)
4. **Test the time slots** by selecting a date in your booking interface
5. **Verify** that times now display correctly (e.g., "9:00 AM", "2:15 PM", "3:30 PM")

## Expected Results After Fix

Before:
- ❌ 2:00 PM AM
- ❌ 2:15 PM AM  
- ❌ 3:30 PM AM
- ❌ 4:15 PM AM

After:
- ✅ 9:00 AM
- ✅ 9:15 AM
- ✅ 2:00 PM
- ✅ 2:15 PM
- ✅ 3:30 PM

## Additional Notes

- The issue only affects the display format, not the underlying time calculations
- No changes are needed to your HTML/JavaScript code
- No database data needs to be migrated
- The fix is backward compatible with existing appointments

## Testing Checklist

After applying the fix:
- [ ] Morning slots (9 AM - 12 PM) show "AM" correctly
- [ ] Afternoon slots (12 PM - 5 PM) show "PM" correctly
- [ ] No double AM/PM indicators appear
- [ ] Existing appointments still load correctly
- [ ] New appointments can be booked successfully
- [ ] Time format is consistent across all slots

## Alternative Format Options

If you prefer different time formats:

### 24-Hour Format
```sql
SELECT to_char(slot_bounds.slot_local, 'HH24:MI')
```
Result: "09:00", "14:15", "15:30"

### With Seconds
```sql
SELECT to_char(slot_bounds.slot_local, 'HH:MI:SS AM')
```
Result: "9:00:00 AM", "2:15:00 PM"

### Lowercase AM/PM
```sql
SELECT to_char(slot_bounds.slot_local, 'HH:MI am')
```
Result: "9:00 am", "2:15 pm"

### Without Leading Zeros
```sql
SELECT to_char(slot_bounds.slot_local, 'FMHH:FMMI AM')
```
Result: "9:0 AM" (note: minutes also lose padding, which may not be desired)

Better option for no leading zeros on hour only:
```sql
SELECT regexp_replace(to_char(slot_bounds.slot_local, 'HH:MI AM'), '^0', '')
```
Result: "9:00 AM", "12:15 PM"

## PostgreSQL Time Format Reference

Common format specifiers for `to_char()`:
- `HH` or `HH12` - Hour (01-12) for 12-hour clock
- `HH24` - Hour (00-23) for 24-hour clock  
- `MI` - Minute (00-59)
- `SS` - Second (00-59)
- `AM` or `PM` - AM/PM indicator (uppercase)
- `am` or `pm` - am/pm indicator (lowercase)
- `FM` - Fill mode prefix (suppresses padding zeros)

For more information, see: https://www.postgresql.org/docs/current/functions-formatting.html
