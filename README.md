# WinCC UserArchive Population Rules (V8.1)

This project updates an **existing** WinCC UserArchive for machine production events.

## Fixed UserArchive schema (must not be changed)

The script only populates/updates this existing column order:

1. ID
2. MachineName
3. OrderNumber
4. SerialNumber
5. OperationNumber
6. PartNumber
7. RuntimeHHMM
8. DowntimeHHMM
9. StartDate
10. StartTime
11. PlannedEndDate
12. PlannedEndTime

Do **not** add/remove/rename/reorder columns and do **not** change field types.

## Merge key and identifier handling

- Composite merge key is exactly:
  `OrderNumber + SerialNumber + OperationNumber + PartNumber`
- Key comparisons trim leading/trailing whitespace in those four fields before matching.
- Identifier fields are treated as strings and are never summed/converted.
- Leading zeros (for example `OperationNumber=0300`) are preserved.

## Merge behavior

When a key match is found:

- `RuntimeHHMM` is the **only** summed field (`HH:MM`, hours may be >24 like `172:03`).
- `DowntimeHHMM` is **not** summed in this script change; existing archive value is kept on merge.
- `StartDate/StartTime` becomes the earliest valid timestamp.
- `PlannedEndDate/PlannedEndTime` becomes the latest valid timestamp.
- `MachineName` is not concatenated; if incoming machine differs, existing machine is kept and a warning is traced.

When no key match is found:

- Incoming row is appended as a new record (same schema/order/types).

## Validation and safety

Before write/merge, incoming row is validated:

- Runtime must be valid `HH:MM`, minutes `00..59`, non-negative hours.
- `StartDate/StartTime` must be a valid date/time in archive format.
- `PlannedEndDate/PlannedEndTime` must be a valid date/time in archive format.

Invalid rows are logged and skipped safely (no silent bad merge, no archive schema changes).

## Manual test checklist

1. **Same key merge**
   Existing `RuntimeHHMM=07:58`, incoming `RuntimeHHMM=02:23` -> merged `10:21`.
   Start should be earliest row; planned end should be latest row.
2. **Same OrderNumber, different SerialNumber** -> separate rows.
3. **Same OrderNumber+SerialNumber, different OperationNumber** -> separate rows.
4. **Same OrderNumber+SerialNumber+OperationNumber, different PartNumber** -> separate rows.
5. **Leading zeros**: `OperationNumber=0300` remains `0300`.
6. **Runtime >24h**: accept `172:03` as duration, not time-of-day.
7. **Bad runtime/date**: invalid runtime/date values are traced and skipped.
