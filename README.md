# Chicago Cityscape Claude Code Skills

[Claude Code](https://claude.ai/code) skills for working with the [Chicago Cityscape](https://www.chicagocityscape.com) codebase and APIs.

## Skills

### chicago-cityscape-api

Describes how to interact with the Chicago Cityscape API. Covers all public endpoints, authentication, and how to obtain an API key.

**Triggers on**: questions about API access, API keys, API endpoints, querying property data programmatically, or integrating Chicago Cityscape data into an application.

**Endpoints documented**:
- Property Report API (`/api/index.php`) — full property data as GeoJSON
- Zoning API (`/api/zoning.php`) — zoning standards for any Chicago zoning class
- Parcels API (`/api/parcels.php`) — parcel geometries filtered by place, boundary, or radius
- Places API (`/api/places.php`) — place boundaries, type discovery, and keyword search

## Installation

Clone this repo and symlink each skill directory into your Claude Code global skills folder:

```bash
git clone https://github.com/ChicagoCityscape/claude-code-skills.git ~/Sites/"Chicago Cityscape Skills"
ln -s ~/Sites/"Chicago Cityscape Skills"/chicago-cityscape-api ~/.claude/skills/chicago-cityscape-api
```

## Usage

Skills are invoked automatically by Claude Code when your request matches a skill's trigger description. You can also invoke them explicitly with a slash command:

```
/chicago-cityscape-api
```
