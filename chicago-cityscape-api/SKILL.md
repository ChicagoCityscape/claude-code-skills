---
name: chicago-cityscape-api
description: >
  Describes how to interact with the Chicago Cityscape API. Use when the user
  asks about API access, API keys, API endpoints, how to query property data
  programmatically, or how to integrate Chicago Cityscape data into their own
  application. Covers the Property Report API, Zoning API, Parcels API, and
  Places API.
allowed-tools:
  - Bash
---

# Chicago Cityscape API Guide

Chicago Cityscape offers several public API endpoints that provide programmatic
access to property data, zoning information, parcel boundaries, and place
boundaries for Chicago and Cook County.

---

## Getting an API Key

**API access requires a Real Estate Pro membership.**

To obtain an API key:

1. Subscribe to the **Cityscape Real Estate Pro** plan at
   `https://chicagocityscape.com/checkout/`
2. Once subscribed, your API key is available in your account at
   `https://chicagocityscape.com/account.php` under the API section.

API keys are 12-character alphanumeric strings tied to your account. They are
verified against your active membership status on every request. If your
subscription lapses, your key will stop working.

---

## Authentication

Pass your API key as a `key` query parameter on every request:

```
https://chicagocityscape.com/api/index.php?key=YOUR_KEY&...
```

Requests without a valid key return:

```json
{ "error": ["Key not provided"] }
```

---

## Endpoints

### 1. Property Report API

Returns the full property report for a given location as GeoJSON, including
zoning, boundaries, parcels, transit access, and optional permit/violation/sales
data.

**Endpoint**: `https://chicagocityscape.com/api/index.php`

**Caching**: Responses are cached for 72 hours.

#### Required Parameters

Provide `key` plus **one** of the following location groups:

| Group | Parameters | Example |
|-------|-----------|---------|
| PIN | `pin=il-cook-{14-digit PIN}` | `pin=il-cook-20222150200000` |
| Full address | `query={full address string}` | `query=121 N La Salle St, Chicago, IL` |
| Address parts | `address`, `city` (default: Chicago), `state` (default: IL), optional `zipcode` | `address=121 N La Salle St&city=Chicago&state=IL` |
| Coordinates | `lat` and `lng` in WGS84/EPSG:4326 | `lat=41.8827&lng=-87.6320` |

#### Optional Parameters

These parameters add data to the response but increase response time:

| Parameter | Description | Notes |
|-----------|-------------|-------|
| `get_permits=true` | Building permits at this address | Chicago only |
| `get_violations=true` | Building violations at this address | Chicago only |
| `get_incentives=true` | Incentives Checker analysis | Requires Real Estate Pro or Enterprise plan |
| `skip_boundaries=true` | Skip surrounding Places lookup | Speeds up response |
| `get_characteristics=true` | Physical characteristics from Cook County Assessor | Requires PIN |
| `get_sales=true` | Property sales from PTAX records | Requires PIN; Illinois only |
| `get_recordings=true` | Property recordings data | Requires PIN; Cook County only |

#### Response Structure

The response is a valid GeoJSON Feature:

```json
{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [-87.6320, 41.8827]
  },
  "properties": {
    "request": {},
    "parcels_intersecting": [],
    "parcels_address": [],
    "parcels_other": [],
    "parcels_same_pin": [],
    "zoning": {},
    "zoning_history": [],
    "boundaries": [],
    "train_stations": {},
    "aro_2021": {},
    "pedestrian_street": {},
    "tsl": {},
    "characteristics": [],
    "permits": [],
    "violations": [],
    "sales": [],
    "recordings": [],
    "elapsed_time": 0.42,
    "cache_id": "...",
    "timestamp": "..."
  }
}
```

Key response properties:

- `parcels_same_pin` — Parcels sharing the same PIN (condos, multi-part lots)
- `parcels_intersecting` — Parcels that intersect the queried point
- `parcels_address` — Parcels matched by address string
- `zoning` — Current Chicago zoning classification and details
- `zoning_history` — Past zoning changes
- `boundaries` — All Places (wards, community areas, ZIP codes, neighborhoods, etc.) containing the property
- `train_stations` — CTA/Metra stations within 1 mile
- `tsl` — Transit-Served Location eligibility and nearby bus routes
- `aro_2021` — Affordable Requirements Ordinance status
- `elapsed_time` — Server-side query time in seconds

#### Example Requests

```bash
# By PIN
curl "https://chicagocityscape.com/api/index.php?pin=il-cook-20222150200000&key=YOUR_KEY"

# By full address
curl "https://chicagocityscape.com/api/index.php?query=121+N+La+Salle+St,+Chicago,+IL&key=YOUR_KEY"

# By coordinates with permits
curl "https://chicagocityscape.com/api/index.php?lat=41.8827&lng=-87.6320&get_permits=true&key=YOUR_KEY"
```

---

### 2. Zoning API

Returns zoning standards and regulations for any Chicago zoning classification.

**Endpoint**: `https://chicagocityscape.com/api/zoning.php`

**Caching**: Responses are cached for 7 days.

#### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `key` | Your API key | `key=YOUR_KEY` |
| `zone_class` | Chicago zoning classification | `zone_class=B1-1.5` |

The `zone_class` lookup is case-insensitive and flexible with formatting
(e.g., `RT-4` and `RT4` both work).

#### Response Structure

```json
{
  "success": true,
  "zone_class": "B1-1.5",
  "data": {
    "far": 1.5,
    "lot_area_per_unit": 1350,
    "lot_area_per_eff_unit": 1350,
    "lot_area_per_sro_unit": 0,
    "max_height": "Varies by lot frontage...",
    "setback_front": "None, unless...",
    "setback_side": "None, unless...",
    "setback_rear": "If property has dwelling units...",
    "mla": null,
    "max_efficiency_units_pct": 15,
    "parking_minimum_ratio": "1",
    "parking_min_ratio_tod": 0.5,
    "parking_min_bike_parking_tod": "1 per unit",
    "common_uses": "Small businesses; one apartment above",
    "max_dwelling_units": null
  },
  "notes": [
    "Responses are cached for 7 days",
    "All measurements are in feet unless otherwise specified",
    "FAR = Floor Area Ratio",
    "MLA = Minimum Lot Area",
    "TOD = Transit-Oriented Development"
  ]
}
```

Key data fields:

- `far` — Floor Area Ratio (max building area relative to lot area)
- `lot_area_per_unit` — Minimum lot area per dwelling unit (sq ft)
- `max_height` — Building height regulations
- `setback_front/side/rear` — Required setbacks from property lines
- `parking_minimum_ratio` — Required parking spaces per unit
- `parking_min_ratio_tod` — Reduced parking minimum in TOD areas
- `common_uses` — Typical uses allowed

#### Error Response

```json
{
  "success": false,
  "error": ["Zone class 'XYZ' not found in database"],
  "zone_class": "XYZ"
}
```

#### Example Request

```bash
curl "https://chicagocityscape.com/api/zoning.php?zone_class=B1-1.5&key=YOUR_KEY"
```

---

### 3. Parcels API

Returns parcel geometries as a GeoJSON FeatureCollection, filtered by a
geographic boundary, radius, or place slug.

**Endpoint**: `https://chicagocityscape.com/api/parcels.php`

**Caching**: Responses are cached for 72 hours.

**Result limit**: Maximum 1,000 results per request.

#### Parameters

| Parameter | Description |
|-----------|-------------|
| `key` | Your API key (required) |
| `slug` | Place slug to filter by (validates ownership for personal places) |
| `bounds_geojson` | URL-encoded GeoJSON polygon for boundary filtering |
| `lat`, `lng`, `radius` | Coordinates + buffer radius in feet for radius search |
| `limit` | Results per page (1–1000, default: 100) |
| `offset` | Pagination offset |
| `prop_class` | Filter by Cook County property class code |
| `zone_class` | Filter by Chicago zoning classification |
| `chicago_owned` | Filter for City of Chicago-owned properties (boolean) |
| `assessed_value_min`, `assessed_value_max` | Assessment value range filter |
| `get_ptax_data` | Include PTAX (property tax) data |

#### Example Request

```bash
curl "https://chicagocityscape.com/api/parcels.php?slug=communityarea-avondale&limit=100&key=YOUR_KEY"
```

---

### 4. Places API (Boundaries)

Returns place boundary data. No API key required. Three sub-methods are
available.

**Endpoints**:
- List all places: `https://chicagocityscape.com/api/places.php?method=allplaces`
- Search by keyword: `https://chicagocityscape.com/api/places.php?query=avondale&method=search`
- Boundary by slug: `https://chicagocityscape.com/php/api.map.php?method=boundary&place=communityarea-avondale`

#### Methods

**`method=place_types`** — Returns all distinct Place types with their row
counts. Use this first to discover what values to pass as `type` to
`allplaces`.

```bash
curl "https://chicagocityscape.com/api/places.php?method=place_types"
```

Example response:
```json
{
  "types": [
    {
      "type": "censusblock",
      "count": "574546",
      "name": "Census Block",
      "description": "..."
    },
    {
      "type": "ward",
      "count": "50",
      "name": "Chicago Ward",
      "description": "Map of current Chicago wards that are based on 2020 Census data that was adopted on May 16, 2022."
    },
    {
      "type": "communityarea",
      "count": "77",
      "name": "Community Area",
      "description": "Chicago's 77 official boundaries created for sociological surveys."
    }
  ],
  "count": 109,
  "notes": [
    "Use the 'type' value with the 'allplaces' method to retrieve all Places of that type",
    "Example: method=allplaces&type=ward"
  ]
}
```

**`method=allplaces`** — Returns all Places of a single type. A `type`
parameter is **required**. Without it the API returns an error, since
fetching all types at once exceeds one million rows and times out.

```bash
curl "https://chicagocityscape.com/api/places.php?method=allplaces&type=ward"
```

Does not return geometry.

**`method=search`** — Search for Places by keyword. Does not return geometry.

```bash
curl "https://chicagocityscape.com/api/places.php?query=avondale&method=search"
```

**`method=boundary`** — Returns a Place's geometry as GeoJSON given its slug.
Use the slug from the list or search results.

```bash
# Community area
curl "https://chicagocityscape.com/php/api.map.php?method=boundary&place=communityarea-avondale"

# Neighborhood
curl "https://chicagocityscape.com/php/api.map.php?method=boundary&place=neighborhood-avondale"
```

---

## Response Headers

All endpoints return:

```
Content-Type: application/json; charset=utf-8
Cache-Control: max-age={seconds}
```

---

## Cook County PIN Format

When passing a PIN to the Property Report API, format it as:

```
il-cook-{14-digit PIN}
```

Example: `il-cook-20222150200000`

The 14-digit PIN follows the format `XX-XX-XXX-XXX-XXXX` (with dashes removed).
PINs may begin with a zero.

---

## Error Handling

| Error | Cause |
|-------|-------|
| `{"error": ["Key not provided"]}` | Missing `key` parameter |
| `{"error": ["Incomplete request (1 or more parameters are missing)"]}` | Missing required location parameter |
| `{"success": false, "error": ["Zone class 'XYZ' not found in database"]}` | Unrecognized zoning class |

---

## Notes

- All APIs track usage asynchronously via a message queue — usage is logged per
  API key and endpoint.
- No explicit rate limiting is enforced, but usage is monitored.
- Cached responses include a `cache_id` and `timestamp` in the response body.
- The `elapsed_time` field in responses shows server-side query execution time
  in seconds.
