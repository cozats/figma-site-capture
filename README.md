# Figma Site Capture Skill for Claude

A specialized Claude skill that fully captures a live website into Figma as editable design layers. It acts as an orchestrator using the Playwright MCP and Figma Remote MCP to crawl a live site, capture pages across breakpoints, and import them into Figma.

## Features

- **Full-Site Discovery**: Automatically finds pages via sitemap or links on the provided URL.
- **Responsive Captures**: Grabs Mobile (375px), Tablet (768px), and Desktop (1440px) layouts.
- **UI State Capturing**: Detects interactive elements (e.g., hover states, modals) and captures them automatically using Playwright scripts.
- **Figma Integration**: Directly uploads and injects the captured layouts into Figma as editable layers.

## Prerequisites

To use this skill, you need Claude with the following MCP servers configured and running:

1. **[Figma Remote MCP](https://www.figma.com/developers/mcp)**: Required to interact with Figma and push designs.
2. **[Playwright MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/playwright)**: Required to browse, render, and execute scripts on the live websites.

## Installation

1. Clone this repository into your local skills directory or where your Claude setup expects skills.
2. Ensure the `.claude-plugin` directory is preserved as it contains the `plugin.json` metadata for the skill.
3. Update `plugin.json` with your own details if you plan to publish or customize it.

## Usage

You can trigger this skill by asking Claude:
- "Capture site to figma [URL]"
- "Import site to figma [URL]"
- "Clone site to figma [URL]"

Claude will guide you through the process, asking you to confirm:
1. The target website URL.
2. Whether to add the captures to an existing Figma file (requires file URL/Key) or create a new one.

Once started, the skill operates autonomously:
- It discovers all pages.
- Defines a capture matrix.
- Executes capturing across viewports.
- Uploads the results into Figma.
- Provides a final report.

## How it works

1. **Discovery**: Runs a script via Playwright to parse `sitemap.xml` and on-page links.
2. **Preflight Script**: Ensures all elements are fully loaded by scrolling down, disabling animations (CSS injection), and disabling CSP headers.
3. **Capture Script**: Injects Figma's internal HTML-to-Design scripts to convert DOM elements to Figma nodes and send them to the Figma MCP endpoint.

## License

MIT
