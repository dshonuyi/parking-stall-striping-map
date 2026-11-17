# Select step: building the Facilities table

The flow is triggered by **When a survey response is submitted** (ArcGIS Survey123 connector). The survey has two repeat sections:

| Repeat | Field | Meaning |
|---|---|---|
| `select_parking_lot_stall_counts` | `parking_lot` | Lot code, for example `P002` |
| | `stalls` | Number of stalls changing in that lot |
| `select_parking_spots` | `field_9` | New space type for the drawn area |
| | `geometry.rings` | Polygon the requester drew, as `[[[lng,lat], ...]]` |

The two repeats are separate lists, so the Select action loops over row numbers and pulls the *n*th entry from each.

## From

One row for each entry in whichever list is longer:

```
range(0, max(
  length(coalesce(triggerOutputs()?['body/feature/repeats/select_parking_lot_stall_counts'], json('[]'))),
  length(coalesce(triggerOutputs()?['body/feature/repeats/select_parking_spots'], json('[]')))
))
```

`coalesce(..., json('[]'))` keeps the flow from failing when a repeat is empty.

## Map (columns)

Inside the Select, `item()` is the row number.

**Parking Lot**
```
coalesce(first(skip(coalesce(triggerOutputs()?['body/feature/repeats/select_parking_lot_stall_counts'], createArray()), item()))?['attributes']?['parking_lot'], '')
```

**Spaces**
```
coalesce(first(skip(coalesce(triggerOutputs()?['body/feature/repeats/select_parking_lot_stall_counts'], createArray()), item()))?['attributes']?['stalls'], '')
```

**Space Type**
```
coalesce(first(skip(coalesce(triggerOutputs()?['body/feature/repeats/select_parking_spots'], createArray()), item()))?['attributes']?['field_9'], '')
```

`first(skip(list, n))` is how you get item *n* from a list in Power Automate's expression language. The `?[...]` operator returns null instead of throwing an error when an index is missing, and the outer `coalesce(..., '')` turns that null into a blank cell.

**Stall Location (map)**
```
concat('MAPLINKSTART', 'https://<user>.github.io/parking-stall-striping-map/#POLYGON((',
  replace(replace(replace(replace(replace(
    string(first(coalesce(first(skip(coalesce(triggerOutputs()?['body/feature/repeats/select_parking_spots'], createArray()), item()))?['geometry']?['rings'], createArray(createArray())))),
  '[[', ''), ']]', ''), '],[', '|'), ',', '%20'), '|', ','),
  '))', 'MAPLINKEND')
```

Step by step, for a ring like `[[-117.2537,32.8640],[-117.2538,32.8639]]`:

| Step | Result |
|---|---|
| `string(ring)` | `[[-117.2537,32.8640],[-117.2538,32.8639]]` |
| remove `[[` and `]]` | `-117.2537,32.8640],[-117.2538,32.8639` |
| `],[` → `\|` (marks point boundaries) | `-117.2537,32.8640\|-117.2538,32.8639` |
| `,` → `%20` (space between lng and lat) | `-117.2537%2032.8640\|-117.2538%2032.8639` |
| `\|` → `,` (comma between points) | `-117.2537%2032.8640,-117.2538%2032.8639` |

That is the WKT point list `lng lat,lng lat,...`, wrapped in `POLYGON((...))`.

## After the Select

1. **Create HTML table** (automatic columns) turns the rows into a `<table>`.
2. **Compose** adds inline CSS (navy header row, borders, padding) and replaces the `MAPLINKSTART...MAPLINKEND` markers with `<a href="...">View on map</a>`.
3. The Compose output goes into the body of the Facilities **Send an email (V2)** action.
