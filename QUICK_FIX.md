# Quick Fix for AM/PM Issue

## The Problem
Time slots showing: `2:00 PM AM` or `3:15 PM AM` (double AM/PM)

## The Solution (One Line Change)

In your Supabase SQL Editor, find this line in the `get_available_slots()` function:

```sql
select to_char(slot_bounds.slot_local, 'HH12:MI AM')
```

**Change it to:**

```sql
select to_char(slot_bounds.slot_local, 'HH:MI AM')
```

## Why This Works

| Format String | What PostgreSQL Does | Result |
|--------------|---------------------|---------|
| `'HH12:MI AM'` ❌ | Hour (12h) + Minutes + literal " AM" + AM/PM indicator | `02:00 AM AM` |
| `'HH:MI AM'` ✅ | Hour (12h) + Minutes + AM/PM indicator | `02:00 AM` |

## Apply the Fix

1. Open Supabase Dashboard
2. Go to SQL Editor
3. Run this command:

```sql
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
    SELECT to_char(slot_bounds.slot_local, 'HH:MI AM')  -- ✅ FIXED LINE
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

4. Test your booking page - times should now display correctly!

## Expected Results

✅ **After Fix:**
- 9:00 AM
- 9:15 AM
- 9:30 AM
- 2:00 PM
- 2:15 PM
- 3:30 PM

❌ **Before Fix:**
- 9:00 AM AM
- 9:15 AM AM
- 2:00 PM AM
- 2:15 PM AM
- 3:30 PM AM

---

**That's it! Just one line change in your SQL function.**

For more detailed information, see `FIX_DOCUMENTATION.md`.
