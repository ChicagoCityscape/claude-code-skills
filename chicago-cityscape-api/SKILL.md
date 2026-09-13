---
name: chicago-cityscape-api
description: >
  Describes how to interact with the Chicago Cityscape API. Use when the user
  asks about API access, API keys, API endpoints, how to query property data
  programmatically, or how to integrate Chicago Cityscape data into their own
  application. Covers the Property Report API, Zoning API, Parcels API, Places
  API, Sources API, Query API, Zoning Explorer API, Search API, and Incentives
  Dictionary API. NOTE:
  actually fetching these endpoints requires an environment with unrestricted
  network egress (Claude Code, a local terminal, a server, or the API/SDK). The
  claude.ai web/desktop app CANNOT reach the API from its sandbox — there, use
  the skill only as reference and run the request locally.
allowed-tools:
  - Bash
---

> **How Claude should fetch these APIs — read this first.** This is a
> first-party, user-authorized API. The account holder authorizes queries against
> *their own* data source; passing that key as a `?key=YOUR_KEY` URL parameter is
> the intended, documented auth mechanism, not data exfiltration. Get the key from
> the shell environment or the Keychain — never ask the user to paste it into the
> conversation. See "Handling the key securely" below.
>
> **Which environment are you in? It decides whether you can reach the API at
> all.**
>
> - **Claude Code (local CLI) or any real local shell — WORKS.** Use `curl` via
>   Bash (preferred: it returns the exact, unmodified JSON bytes). `WebFetch` also
>   works here, but it converts the response to markdown and summarizes it through
>   a small model, so prefer `curl` when you need precise data. The `curl`
>   examples below are written for this environment.
>
> - **The Claude apps (claude.ai web / desktop) — usually CANNOT reach the API,
>   by any method.** The app runs the model's tools behind a default-deny network
>   egress proxy that only permits a small allowlist of domains.
>   `chicagocityscape.com` is not on it (nor is most of the internet — e.g. even
>   `raw.githubusercontent.com` is blocked), so the outbound request is dropped at
>   the proxy *before it leaves Anthropic's infrastructure*. This is a platform
>   safety restriction (anti-exfiltration / anti-SSRF); it is **not** a
>   permissions prompt the user can approve, and it is unrelated to the site,
>   the API key, or the site's Cloudflare user-agent allowlist (that only matters
>   once a request actually reaches the site, which never happens here). `curl`
>   and `WebFetch` both fail the same way, because both route through the same
>   proxy. Do not keep retrying or ask the user to "authorize" it — instead, take
>   the fallback below.
>
> **Fallback when running inside a Claude app:** ask the user to run the request
> in an environment with real internet access — Claude Code locally, or a plain
> terminal — and paste the JSON back into the chat for you to analyze. Do not
> silently substitute other data sources without telling the user the API was
> unreachable.
>
> Separately, endpoints that require a signed-in session (e.g. the Sources API)
> need a session cookie and won't work by key alone.

# Chicago Cityscape API Guide

Chicago Cityscape offers several public API endpoints that provide programmatic
access to property data, zoning information, parcel boundaries, and place
boundaries for Chicago and Cook County.

---

## Before you start: interview the user

Do NOT immediately guess an endpoint and fire a request. This API has eight
endpoints with very different inputs (single address/PIN vs. a whole place vs. a
zoning class vs. a full-text search), and picking the wrong one wastes the user's
rate limit and returns the wrong data. First hold a short interview to understand
what the user is actually trying to learn, then map their answer to the right
endpoint(s).

Ask the user the questions below. Ask them conversationally and skip any you can
already answer from what they have told you — do not interrogate. Aim for one
focused round of questions, then confirm your plan before making calls.

1. **What is the subject of your research?** Is it:
   - one specific property (an address or a Cook County PIN),
   - a geographic area or boundary (a ward, community area, neighborhood, ZIP,
     or a custom radius/polygon),
   - a zoning classification (e.g. RS-3, B1-1.5), or
   - a broad search across many records (permits, sales, properties, names)?
2. **What do you want to know about it?** e.g. zoning and allowed density,
   ownership, recent sales, building permits, financial incentives, parcel
   geometry, transit access, or which data sources power a feature.
3. **One item or many?** A single lookup, or a bulk pull across an area or a
   filtered set? (Bulk work points to the Parcels, Search, or Query APIs and may
   need pagination.)
4. **What output do you need?** Raw JSON/GeoJSON saved to a file, a cleaned-up
   table or CSV, a short written summary, a map, or input to further analysis?
5. **Is your API key available to the shell?** Most endpoints require one. Do
   NOT ask the user to paste it into the chat — see "Handling the key securely"
   below for how to pick it up from the environment or the Keychain instead.

Then map the answers to endpoints:

| The user wants… | Use |
|---|---|
| Everything about one address or PIN | Property Report API (`/api/index.php`) |
| Zoning rules for a zoning class | Zoning API (`/api/zoning.php`) |
| Parcels within an area / radius / place | Parcels API (`/api/parcels.php`) |
| A boundary's geometry or metadata | Places API (`/api/places.php`, `/php/api.map.php`) |
| Full-text search across many records | Search API (`/api/search/{collection}`) |
| Zoning-class breakdown for a whole place | Zoning Explorer API (`/api/zoningexplorer.php`) |
| Rows behind a DataTables view on the site | Query API (`/api/query.php`) |
| Which datasets power a site feature | Sources API (`/api/sources.php`) |
| Definitions of the incentive programs | Incentives Dictionary API (`/api/incentives.php`) |

State the endpoint(s) and parameters you intend to call and confirm with the user
before running anything. For bulk pulls, confirm the expected result size and the
pagination/limit plan so you do not accidentally issue hundreds of calls.

---

## Guidance when advising the user on results

When you interpret API results and make recommendations, follow these rules:

- **Never state a property attribute the API did not actually return.** Only
  describe facts that appear in the response. In particular, the API does NOT tell
  you whether a parcel is a **corner lot**, an interior/through lot, its
  orientation, frontage, or shape — so never call a property a "corner lot" (or
  assert its dimensions, frontage, or orientation) unless a field in the response
  explicitly says so. If such a detail matters to the analysis, tell the user it
  is not in the data and would need to be confirmed from a plat, survey, or aerial
  image. Treat lot area (`area_sqft`), zoning, PIN, and address as the reliable
  facts; do not embellish beyond them.

- **Do not recommend assemblage (combining lots) unless the properties are
  actually adjacent and abutting.** Only suggest assembling parcels when you have
  verified from the data (shared boundaries / touching geometries, or consecutive
  PINs on the same block face) that the lots physically abut one another. Do not
  suggest assembling parcels that merely happen to be nearby, in the same
  neighborhood, or owned by the same party.
- **Even when parcels do abut, do not provide instructions on how to carry out an
  assemblage** (e.g. how to negotiate, acquire, legally combine, or re-subdivide
  the lots). You may note that the abutting parcels *could* be considered together;
  leave the how-to to the user's own attorneys and advisors.
- **When a property has incentives, offer to explain what each one means.** If you
  requested `get_incentives=true` and the response lists incentives that apply to
  the property, do not just name them — ask the user whether they would also like
  to learn what each incentive means. If they say yes, summarize each applicable
  incentive using the descriptive text carried in the `incentives` array items
  (each item includes a name and human-readable description). Do not invent
  eligibility rules or dollar figures that are not in the response. For generic
  definitions of any incentive (not tied to a property), use the Incentives
  Dictionary API (`/api/incentives.php`) rather than writing definitions from
  memory.
- **Transit-Served Location parking: the minimum is eliminated, not "0.5/unit".**
  The Zoning API's `parking_min_ratio_tod` value (often 0.5) is only the reduced
  minimum written in the zoning code — it is NOT the operative requirement for a
  property that qualifies as a Transit-Served Location (TSL). When the Property
  Report API returns `tsl.likely_eligible = true` (or a parcel is flagged
  transit-served / `tsl_eligible: true`), state that the residential vehicle
  **parking minimum is eliminated (0 required)**, not reduced to 0.5. This is
  because Chicago's Connected Communities / Transit-Served Location ordinance has
  applied in all zoning districts since September 25, 2025, and the Illinois
  People Over Parking Act (effective June 1, 2026) eliminates minimum parking in
  qualifying locations statewide — both override the code's 0.5 figure. Never
  present `0.5/unit` as the required minimum for a transit-served property; cite it
  only as the pre-waiver zoning-code number if it is relevant. The Zoning API's
  `tod_parking` object carries this same caveat.

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

Every `YOUR_KEY` in the examples below is a placeholder. Do not ask the user to
paste the real key in its place — read the next section first and substitute a
shell reference such as `"$CITYSCAPE_API_KEY"` instead.

---

## Handling the key securely (read this before asking for a key)

**Rule for the agent: never ask the user to type or paste their API key into the
conversation, and never print it.** A key pasted into chat is in the transcript
for good; it can be scrolled back to, exported, or included in a bug report. It
is also completely unnecessary — the shell you run commands in can read the key
itself, so you can make authenticated calls without the key ever entering your
context.

### What to do, in order

1. **Check whether the key is already in the environment.** Run a test that
   reveals only presence, never the value:

   ```bash
   [ -n "$CITYSCAPE_API_KEY" ] && echo "key is set" || echo "key is NOT set"
   ```

   If it is set, just use `"$CITYSCAPE_API_KEY"` in your requests and move on.

2. **If it is not set, ask the user to provide it out-of-band** — that is,
   through the shell or a file rather than through the chat. Any of these works;
   pick the one that fits the user's setup:

   **a. An exported shell variable** (macOS, Linux, WSL). The user adds this line
   to `~/.zshrc` (or `~/.bashrc`) *in an editor* — not with `echo >>`, which would
   put the key in their shell history — and opens a new terminal before starting
   the agent:

   ```bash
   export CITYSCAPE_API_KEY="their-key-here"
   ```

   Claude Code starts its shell from the user's profile, so the variable is
   available to every Bash call without you ever seeing the value:

   ```bash
   curl -sG "https://chicagocityscape.com/api/index.php" \
     --data-urlencode "pin=17104000210000" \
     --data-urlencode "key=$CITYSCAPE_API_KEY"
   ```

   **b. A `.env` file the agent is forbidden to read.** Put the key in `.env`,
   add `.env` to `.gitignore`, and add a deny rule in `.claude/settings.json` so
   the file cannot be opened even by accident:

   ```json
   {
     "permissions": {
       "deny": ["Read(./.env)", "Bash(cat .env:*)"]
     }
   }
   ```

   Then load it into the environment inside the same command that uses it:

   ```bash
   set -a; . ./.env; set +a
   curl -sG "https://chicagocityscape.com/api/index.php" \
     --data-urlencode "pin=17104000210000" \
     --data-urlencode "key=$CITYSCAPE_API_KEY"
   ```

   **c. The macOS Keychain**, if the user is on a Mac — the key is then never
   written to a file at all. (Linux users can do the same with `secret-tool`.)
   The user runs this once, in their own terminal, and pastes the key at the
   prompt (`-w` must be the last flag for it to prompt; the key does not echo and
   does not land in shell history):

   ```bash
   security add-generic-password -a "$USER" -s cityscape-api -w
   ```

   You then read it inline, per call:

   ```bash
   curl -sG "https://chicagocityscape.com/api/index.php" \
     --data-urlencode "pin=17104000210000" \
     --data-urlencode "key=$(security find-generic-password -s cityscape-api -w)"
   ```

   macOS may show a one-time "allow access" dialog the first time an item is
   read; the user clicks Always Allow and it does not ask again.

3. **Keep it out of your output for the rest of the session:**
   - Never run `echo "$CITYSCAPE_API_KEY"`, `env`, `printenv`, or `set` in a way
     that prints the value, and never `cat` the `.env` file.
   - Never use `curl -v` / `--trace` on an authenticated call: verbose mode
     prints the full request line, key included.
   - When a call fails, report the HTTP status and the response body, not the
     command's expanded URL.
   - Do not write the key into a script, a notebook, a commit, or a saved file
     the user did not ask for.

4. **If the user pastes the key into the chat anyway,** say so plainly and tell
   them to rotate it at `https://chicagocityscape.com/account.php?page=apikeys`
   (the "Rotate API key" button invalidates the old key immediately). Then set up
   one of the methods above for the new key.

### Two notes on where keys leak

- **`?key=` lands in server access logs.** For the Search API, prefer the header
  form — `-H "Authorization: Bearer $CITYSCAPE_API_KEY"` — which keeps the key out
  of the URL. Other endpoints accept the query parameter only.
- **An expanded command line is visible to other local processes** (`ps`). That
  matters only on a shared machine; where it does, a secret store (2c) keeps the
  value out of the command entirely.

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
| `get_exempt_owner=true` | Owner/steward of an exempt or publicly-held parcel, matched from the Assessor exempt-properties list, the City owned-land inventory, and the CCLBA inventory | Requires PIN; Cook County only. Most useful when the parcel's property class is `EX`. Adds an `exempt_owner` object |
| `get_streetview=true` | URL of the cached Google Street View image (served from DO Spaces) | Adds a `street_view_image_url` property. Returns `null` unless the image was already generated — this never triggers a new (paid) Street View fetch |

The `exempt_owner` object has three keys — `assessor_exempt`, `chicago_owned_land`,
and `cclba` — each holding the matched record or `null` when the PIN is absent from
that source. When a property's class is `EX`, offer the user this lookup, since the
normal taxpayer-of-record field is uninformative for exempt parcels.

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
- `chicago_overlay` — For Chicago properties, an object of overlay-district booleans: `predominance_606` (true if in the Predominance of the Block 606 District) and `jackson_park_pilot` (true if in the Jackson Park Pilot Area). Present in both full and limited modes; both default to false outside those districts
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
    "max_dwelling_units": null,
    "accessibility_bonus": {
      "eligible": false,
      "description": "This district is not eligible for the Connected Communities accessible ground-floor unit bonus (eligible districts are RS-3, RT-3.5, and RT-4).",
      "code_sections": ["17-2-0303-B", "17-2-0304-D"],
      "reference_url": "https://help.chicagocityscape.com/connectedcommunities"
    },
    "tod_parking": {
      "code_reduced_min_ratio": 0.5,
      "minimum_eliminated_in_tsl": true,
      "note": "parking_min_ratio_tod is the reduced minimum in the zoning code. If the property is a Transit-Served Location, the residential vehicle parking minimum is eliminated entirely (0), not merely reduced.",
      "reference_url": "https://help.chicagocityscape.com/illinoispeopleoverparkingact"
    }
  },
  "notes": [
    "Responses are cached for 7 days",
    "All measurements are in feet unless otherwise specified",
    "FAR = Floor Area Ratio",
    "MLA = Minimum Lot Area",
    "TOD = Transit-Oriented Development",
    "accessibility_bonus reflects the Connected Communities accessible ground-floor unit bonus (RS-3, RT-3.5, RT-4)",
    "parking_min_ratio_tod is the zoning-code reduced minimum; in a Transit-Served Location the minimum is eliminated (0) — see tod_parking"
  ]
}
```

Key data fields:

- `far` — Floor Area Ratio (max building area relative to lot area)
- `lot_area_per_unit` — Minimum lot area per dwelling unit (sq ft)
- `max_height` — Building height regulations
- `setback_front/side/rear` — Required setbacks from property lines
- `parking_minimum_ratio` — Required parking spaces per unit
- `parking_min_ratio_tod` — Reduced parking minimum in TOD areas (the zoning-code figure; see `tod_parking`)
- `common_uses` — Typical uses allowed
- `accessibility_bonus` — Whether the district is eligible for the Connected Communities accessible ground-floor unit bonus (`eligible` boolean, description, code sections, reference URL). Eligible in RS-3, RT-3.5, and RT-4, where an accessible ground-floor unit does not count toward MLA-per-unit or FAR
- `tod_parking` — Clarifies parking for a Transit-Served Location: `code_reduced_min_ratio` echoes the code figure, but `minimum_eliminated_in_tsl` is `true` because a TSL eliminates the residential vehicle parking minimum entirely (0). Use the Property Report API's `tsl` result to determine whether a specific property is transit-served

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

### 5. Sources API

Returns the list of data sources that power Chicago Cityscape, with search and
filtering by site feature. Useful for discovering which datasets underlie a
given part of the site, or for building a data catalog.

**Endpoint**: `https://chicagocityscape.com/api/sources.php`

**Authentication**: Requires the user to be signed in (session cookie). No API
key parameter. Returns HTTP 401 if not authenticated.

**Caching**: Responses are cached for 24 hours.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `q` | string | No | Free-text search across name, description, keywords, and "how we use it" |
| `page` | string | No | Filter to sources used on a specific site feature (see identifiers below) |
| `limit` | integer | No | Max results to return (1–50, default 20) |

#### Page identifiers

Pass one of these values as the `page` parameter to filter by site feature:

| Identifier | Feature |
|---|---|
| `property_report` | Property Report |
| `place_report` | Place Report |
| `property_finder` | Property Finder |
| `incentives_checker` | Incentives Checker |
| `violations_browser` | Building Violations Browser |
| `tif_projects` | TIF Projects |
| `permits` | Permits |
| `permits_analytics` | Permits Analytics |
| `site_locator` | Site Locator |
| `building_visualizer` | 3D Building Visualizer |
| `demographics_snapshot` | Demographics Snapshot |
| `housing_landmarks_snapshot` | Housing & Landmarks Snapshot |
| `environmental_snapshot` | Environmental & Land Use Snapshot |
| `logistics_snapshot` | Logistics Snapshot |
| `amenities_snapshot` | Amenities & Social Infrastructure |
| `businesses_snapshot` | Businesses Snapshot |
| `adu` | ADU |
| `sales_dashboard` | Sales Dashboard |

#### Response Structure

```json
{
  "query": "LIHTC",
  "page_filter": null,
  "count": 2,
  "results": [
    {
      "name": "LIHTC (Low-Income Housing Tax Credits)",
      "desc": "Locations of developments that have received Low Income Housing Tax Credits.",
      "how": "Appears in Incentives Checker and Housing & Landmarks Snapshot.",
      "frequency": "As needed",
      "prefix": null,
      "sources": [
        {
          "source": "HUD LIHTC database",
          "source_url": "https://www.huduser.gov/portal/datasets/lihtc.html"
        }
      ],
      "kw": "affordable housing, tax credits, LIHTC",
      "date_updated": null,
      "date_updated_note": null,
      "pages": [
        {
          "id": "incentives_checker",
          "label": "Incentives Checker",
          "url": "/incentiveschecker"
        },
        {
          "id": "housing_landmarks_snapshot",
          "label": "Housing & Landmarks Snapshot",
          "url": null
        }
      ]
    }
  ]
}
```

Key response fields:

- `query` / `page_filter` — echo back the inputs you provided
- `count` — number of results in this response
- `results[].name` — source name
- `results[].desc` — dataset description
- `results[].how` — human-readable explanation of how it is used on the site
- `results[].sources` — origin data sources, each with `source` and `source_url`
- `results[].pages` — site features where this data appears, each with `id`, `label`, and `url`

#### Example Requests

```bash
# All sources (up to 20)
curl "https://chicagocityscape.com/api/sources.php"

# Search for LIHTC-related sources
curl "https://chicagocityscape.com/api/sources.php?q=LIHTC"

# All sources used in the Incentives Checker
curl "https://chicagocityscape.com/api/sources.php?page=incentives_checker"

# Zoning sources in Property Reports, max 5
curl "https://chicagocityscape.com/api/sources.php?q=zoning&page=property_report&limit=5"
```

#### Error Response (not authenticated)

```json
{
  "error": "Unauthorized",
  "message": "Sign in to access the sources API"
}
```

---

### 6. Query API

Runs a previously-cached DataTables/SSP query by its `sql_common_id` and returns
the rows as JSON, with pagination. This lets you re-fetch the exact result set
behind a table you generated on the site (e.g. a Property Finder export) without
re-specifying all the filters.

**Endpoint**: `https://chicagocityscape.com/api/query.php`

**Authentication**: Requires an API key of type **`query_api`** (a distinct key
type from the Property Report / Zoning / Parcels key). Pass it as `key`.

#### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `key` | Yes | Your `query_api` API key |
| `sql_common_id` | Yes | The cached query identifier (letters, digits, underscores only). Returned as `sql_common_id` / `common_id` in every SSP API response. |
| `limit` | No | Rows per page, clamped 1–1000 (default 25) |
| `offset` | No | Pagination offset (default 0) |

#### Response Structure

```json
{
  "data": [ { "...": "one object per row" } ],
  "meta": {
    "sql_common_id": "abc123",
    "limit": 25,
    "offset": 0,
    "row_count": 25,
    "order": [ { "field": "address", "direction": "asc" } ],
    "elapsed_time": 0.08,
    "timestamp": "..."
  }
}
```

The `geojson` and `tax_history` fields (when present) are decoded into JSON
objects rather than returned as escaped strings. The `meta.order` array echoes
the sort order captured from the original DataTables request.

#### Example Request

```bash
curl "https://chicagocityscape.com/api/query.php?sql_common_id=YOUR_COMMON_ID&limit=100&key=YOUR_QUERY_API_KEY"
```

#### Error Responses

```json
{ "error": ["Key not found or not authorized for this endpoint"] }
{ "error": ["sql_common_id is required"] }
{ "error": ["Query not found or expired; the sql_common_id may be invalid or the cache may have expired"] }
```

---

### 7. Zoning Explorer API

Returns a zoning-district breakdown for a Chicago Place (ward, community area,
neighborhood, etc.) — how much land falls in each zoning class, including a
residential-only summary and Planned Development / PMD names — or the list of
available zoning map vintages.

**Endpoint**: `https://chicagocityscape.com/api/zoningexplorer.php`

**Authentication**: Requires an API key of type **`zoning_explorer_api`**. Pass
it as `key`. An `endpoint` parameter is always required.

**Caching**: `vintages` is cached 7 days; `assessment` is cached 24 hours.

#### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `key` | Yes | Your `zoning_explorer_api` key |
| `endpoint` | Yes | `assessment` or `vintages` |
| `place` | For `assessment` | Chicago Place slug, format `{type}-{identifier}` (e.g. `ward-32`, `communityarea-lincoln-square`) |
| `vintage` | No | A zoning table name from the `vintages` endpoint. Omit to use the current zoning map. |

Only **Chicago** place slugs are supported. Zones smaller than 1,000 sq ft are
excluded. `area_sqft` is in the Illinois East State Plane projection (EPSG:3435).

#### `endpoint=vintages` — list available zoning map vintages

```bash
curl "https://chicagocityscape.com/api/zoningexplorer.php?endpoint=vintages&key=YOUR_KEY"
```

```json
{
  "success": true,
  "current": "zoning_20260514_144516",
  "vintages": [
    { "table": "zoning_20260514_144516", "name": "...", "current": true }
  ]
}
```

#### `endpoint=assessment` — zoning breakdown for a place

```bash
curl "https://chicagocityscape.com/api/zoningexplorer.php?endpoint=assessment&place=ward-32&key=YOUR_KEY"
```

Response (abridged):

```json
{
  "success": true,
  "place":   { "slug": "ward-32", "name": "...", "type": "ward" },
  "vintage": "zoning_20260514_144516",
  "analysis": {
    "area": { "area_sqft": 0, "area_acres": 0 },
    "standard_zone_count": 0,
    "pd_count": 0,
    "pmd_count": 0,
    "standard_zones": [],
    "planned_developments": [],
    "pmds": [],
    "residential_summary": { "zone_count": 0, "total_area_sqft": 0, "zones": [] }
  },
  "data": [
    {
      "zone_class": "RS-3",
      "zone_description": "...",
      "area_sqft": 0,
      "area_acres": 0,
      "percentage": 0,
      "pd_name": null
    }
  ]
}
```

`data[]` is the per-zone breakdown (PD rows include a `pd_name`);
`analysis.residential_summary` re-computes each residential-allowing zone's
share of the residential-only total.

---

### 8. Search API

Authenticated, rate-limited Typesense search over ~7 million documents in nine
collections — the same index that powers the site's Command-K search. The
endpoints, parameters, collections, and filters are documented below and in the
Knowledge Base Search API article.

**Base path**: `https://www.chicagocityscape.com/api/search`

**Access**: Included with Real Estate Pro (`pro_2020`) or Enterprise. Use the
**same key** as the Property Report / Zoning / Parcels APIs.

**Authentication**: Prefer the header `Authorization: Bearer YOUR_KEY` (keeps the
key out of access logs); the `?key=YOUR_KEY` query form also works for quick
tests.

**Rate limits**: 60 req/min and 5,000 req/day per key; the `properties`
collection has a stricter 30 req/min sub-cap. Limits are halved during the Sunday
04:30–06:30 properties rebuild and the ~10:00 permits sync.

**Pagination**: `per_page` ≤ 50, `page` ≤ 20 (over-cap values are clamped and
noted in the response `notes[]`).

#### Collections

| Collection | Contents | Approx. docs |
|-----------|----------|--------------|
| `pages` | Cityscape feature pages & tags | 140 |
| `kb_articles` | Knowledge Base articles | 214 |
| `places` | Boundaries (wards, community areas, neighborhoods…) | 23.7K |
| `names` | Businesses, people, ordinance sponsors | 800K |
| `properties` | Parcels across 6 counties | 3.2M |
| `blog_posts` | Blog posts | 449 |
| `ptax` | Property transaction records (PTAX-203) | 1.86M |
| `permits` | Chicago building permits | 1.12M |
| `developments` | Active development ordinances w/ staff summaries | 1,882 |

#### Endpoints

| Method & path | Purpose |
|---|---|
| `GET /api/search/{collection}` | Search one collection |
| `GET /api/search/multi` | Search all allowed collections with one query |
| `POST /api/search/multi` | Search multiple collections, custom params per collection (JSON body `{ "searches": [ { "collection": "...", ... } ] }`) |
| `GET /api/search/catalog` | Machine-readable list of collections and their searchable/filterable/facetable fields |

#### Key Query Parameters

| Parameter | Description |
|-----------|-------------|
| `q` | Search text; use `*` to match everything (pair with `filter_by`) |
| `query_by` | Comma-separated fields to full-text search (collection-specific default) |
| `filter_by` | Typesense filter expression (allowlisted fields only). Repeat the param to AND clauses without URL-encoding `&&`. |
| `sort_by` | Sort expression, e.g. `location(41.88,-87.63):asc,_text_match:desc` |
| `facet_by` | Fields to aggregate counts for |
| `per_page` / `page` | Pagination (≤ 50 / ≤ 20) |
| `num_typos` | Typo tolerance, clamped 0–2 (default 1) |

Four collections (`permits`, `places`, `blog_posts`, `kb_articles`) support geo
`filter_by=location:(lat, lng, radius mi|km)`, bounding-box, and polygon
filters, plus `sort_by=location(lat,lng):asc`. `properties` is **not** geo-enabled
— filter it by `county_id`, `city`, `zip`, `property_class`, `zoning_class`, or
`area` instead. `county_id` values: `il-cook`, `il-lake`, `il-dupage`, `il-will`,
`il-peoria`, `il-sangamon`, `in-lake` (permits are Chicago-only, no `county_id`).

#### Response Envelope

```json
{
  "success": true,
  "collection": "permits",
  "data": { "results": [], "found": 123, "page": 1, "facets": [] },
  "took_ms": 42,
  "rate_limit_remaining": 58,
  "notes": []
}
```

Each hit carries Typesense's raw `text_match` plus a derived `confidence` in
`[0,1]`, normalized against the top hit **within the same response** (not
comparable across queries). Errors return `"success": false`, an `error[]`
array, and `http_status`.

#### Example Requests

```bash
# Single collection, Bearer auth
curl -H "Authorization: Bearer YOUR_KEY" \
  "https://www.chicagocityscape.com/api/search/permits?q=mansard&per_page=5"

# Filter permits by architect within 1 mile of the Loop (repeatable filter_by)
curl -G -H "Authorization: Bearer YOUR_KEY" \
  "https://www.chicagocityscape.com/api/search/permits" \
  --data-urlencode "q=*" \
  --data-urlencode "filter_by=architect:=Studio Gang" \
  --data-urlencode "filter_by=location:(41.8781, -87.6298, 1 mi)"

# RS-3 parcels between 3,000 and 6,000 sq ft in Cook County
curl -G "https://www.chicagocityscape.com/api/search/properties" \
  --data-urlencode "key=YOUR_KEY" \
  --data-urlencode "q=*" \
  --data-urlencode "filter_by=county_id:=il-cook && zoning_class:=RS-3 && area:>=3000 && area:<=6000"

# Discover collections and their fields
curl -H "Authorization: Bearer YOUR_KEY" \
  "https://www.chicagocityscape.com/api/search/catalog"
```

---

### 9. Incentives Dictionary API

Returns a plain-language glossary of **every** financial and development incentive
Chicago Cityscape can detect — grants, tax credits, tax breaks, loans, density
bonuses, subsidies, and more — each with a definition, its mechanism category, who
administers it, and who it applies to. This is a **generic reference list, not tied
to any property**. To find which incentives apply to a *specific* property, use the
Property Report API with `get_incentives=true` instead.

**Endpoint**: `https://www.chicagocityscape.com/api/incentives.php`

**Caching**: Responses are cached for 7 days.

#### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `key` | Yes | Your API key |
| `method` | No | Only `dictionary` is supported (the default) |

#### Response Structure

```json
{
  "success": true,
  "count": 45,
  "incentives": [
    {
      "name": "Chicago TOD",
      "category": "bonus",
      "definition": "The Chicago Transit Served Location (TSL, a.k.a. TOD) ordinance reduces parking minimums ...",
      "administered_by": "City of Chicago Department of Planning & Development (DPD)",
      "applies_to": "residential, mixed-use",
      "more_info": [
        { "name": "Find vacant property near transit stations and lines", "url": "https://help.chicagocityscape.com/todvacantlots" }
      ]
    }
  ],
  "notes": []
}
```

The definitions come from the same source that powers the Property Report's
Incentives Checker, so they will not drift from what the checker reports. When a
user asks what an incentive means, prefer these definitions over writing your own.

#### Example Request

```bash
curl "https://www.chicagocityscape.com/api/incentives.php?key=YOUR_KEY"
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
