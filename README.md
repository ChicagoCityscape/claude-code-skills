# Chicago Cityscape plugin for Claude

A Claude plugin that teaches Claude how to use the [Chicago Cityscape](https://www.chicagocityscape.com) APIs to research real estate in Chicago and Cook County: property reports, zoning rules, parcel boundaries, building permits, pending permits, financial incentives, and place boundaries such as wards and community areas.

## What's included

### Skill: chicago-cityscape-api

Documents all ten public Chicago Cityscape API endpoints: Property Report, Zoning, Parcels, Places, Sources, Query, Zoning Explorer, Search, Incentives Dictionary, and Pending Permits. Before calling anything, Claude asks a few short questions about what you are researching, then picks the right endpoint.

**Triggers on**: questions about API access, API keys, API endpoints, querying property data programmatically, or integrating Chicago Cityscape data into an application. You can also invoke it directly with `/chicago-cityscape-api`.

## Requirements

- A Chicago Cityscape account with an API key. Get one at [chicagocityscape.com/account.php?page=apikeys](https://chicagocityscape.com/account.php?page=apikeys).
- An environment with open internet access, such as Claude Code or a local terminal. The claude.ai web and desktop apps run tools behind a network allowlist that does not include chicagocityscape.com. There, Claude uses the skill as reference only and asks you to run the request locally.

## What this plugin runs, sends, and fetches

- **Runs**: `curl` commands through the Bash tool, which Claude Code asks you to approve.
- **Sends**: requests only to `https://chicagocityscape.com` (and `www.chicagocityscape.com`). Each request carries your query parameters (an address, PIN, place, zoning class, or search term) and your Chicago Cityscape API key.
- **Where the key comes from**: the skill tells Claude never to ask you to paste the key into the chat. Claude reads it from the `CITYSCAPE_API_KEY` environment variable, from a `.env` file loaded inside the same command, or from the macOS Keychain (item `cityscape-api`), and never prints it. The key goes only to chicagocityscape.com, the service that issued it.
- **Stores**: nothing. The plugin has no hooks, MCP servers, or background processes, and saves no data outside the files you ask Claude to write.

See the [Chicago Cityscape privacy policy](https://help.chicagocityscape.com/privacypolicy) for how chicagocityscape.com handles API requests.

## Installation

In Claude, find **Chicago Cityscape** in the plugin directory and add it.

To install from this repository in Claude Code instead:

```bash
git clone https://github.com/ChicagoCityscape/claude-code-skills.git
claude --plugin-dir ./claude-code-skills
```

## License

MIT. See [LICENSE](LICENSE).
