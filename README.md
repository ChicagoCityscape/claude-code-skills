# Chicago Cityscape plugin for Claude

A Claude plugin that adds the [Chicago Cityscape](https://www.chicagocityscape.com) connector, so Claude can read Property Reports for any property in 13 counties in Illinois, Indiana and Wisconsin: zoning and the uses it allows, property class and building characteristics, property taxes, building permits, violations, sales, ownership, development incentives and nearby transit.

Want Claude Code to call the Chicago Cityscape APIs directly with your API key? Use the **Chicago Cityscape API** plugin instead. Its source is [ChicagoCityscape/claude-code-api-skill](https://github.com/ChicagoCityscape/claude-code-api-skill).

## What's included

### Connector: Chicago Cityscape (MCP server)

A remote MCP server at `https://www.chicagocityscape.com/mcp`. You sign in once with your Chicago Cityscape account. Claude can then:

- look up a property by address or PIN and read its Property Report, then ask for one part in depth: every incentive, the uses the zoning allows, a map, a building permit, or (Cook County) assessment appeals and property tax exemptions
- read up to 5 Property Reports in one request
- compare up to 30 properties by location, lot size and zoning
- list the properties in your My Faves and My Notes
- add a property to My Faves when you ask

It works in claude.ai, the desktop app, Cowork and Claude Code. How much of each report you see depends on your membership.

Counties covered: Cook, Lake, DuPage, Will, Kane, Kendall, McHenry, Peoria, Sangamon and Winnebago in Illinois; Lake in Indiana; Milwaukee and Dane in Wisconsin.

## Requirements

A Chicago Cityscape account. Sign up at [chicagocityscape.com](https://www.chicagocityscape.com). No API key is needed.

## What this plugin runs, sends, and fetches

- **Runs**: nothing on your machine. The plugin has no skills, hooks, scripts or background processes.
- **Sends**: requests only to `https://www.chicagocityscape.com/mcp`, with each tool's arguments (an address, PIN or county) and an OAuth token from your sign-in. The connector never sees your password or payment details.
- **Changes**: the only change it can make to your account is adding a property to My Faves when you ask. You can disconnect it at any time in Chicago Cityscape's My Account.
- **Stores**: nothing on your machine.

See the [Chicago Cityscape privacy policy](https://help.chicagocityscape.com/privacypolicy) for how chicagocityscape.com handles your data.

## Installation

In Claude, find **Chicago Cityscape** in the plugin directory and add it, then sign in with your Chicago Cityscape account when Claude asks.

You can also add the connector by itself: in Claude, go to **Settings**, then **Connectors**, and add a custom connector with the URL `https://www.chicagocityscape.com/mcp`.

## License

MIT. See [LICENSE](LICENSE).
