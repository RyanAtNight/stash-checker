# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Stash Checker is a Tampermonkey/Violentmonkey userscript that runs on 50+ adult websites and checks if scenes/performers/studios are present in the user's local [Stash](https://github.com/stashapp/stash) instance. It displays checkmarks with metadata tooltips when matches are found.

**Stack:**
- TypeScript 5.9.3 (strict mode)
- Webpack 5 with webpack-userscript plugin
- SCSS for styling
- Greasemonkey API for browser storage and permissions

## Build & Development Commands

### Development Setup
```bash
# Install dependencies
npm ci

# Start development server (port 8080)
npm run dev
```

**Important development workflow:**
1. Run `npm run dev` to start webpack dev server
2. Install `dist/index.dev.proxy.user.js` in Tampermonkey (once)
   - This is a loader that loads the actual dev build
3. DO NOT install `dist/index.dev.user.js` - it's loaded automatically
4. Use [Chrome livereload extension](https://chrome.google.com/webstore/detail/jnihajbhpnppcggbcgedagnkighmdlei) for hot reload
5. If you change `metadata.js`, restart the webpack server and reinstall the proxy script

### Production Build
```bash
# Build final userscript
npm run build
```

Output files:
- `dist/index.prod.user.js` - Final production userscript
- `dist/index.prod.meta.js` - Version metadata only (for update checks)

### Dependency Management
```bash
# Check for dependency updates
npm run updates
```

## Architecture

### Core Flow

1. **Site Detection** (`stashChecker.ts`) - Identifies website via `window.location.host` and loads site-specific configuration
2. **Element Discovery** (`check.ts` + `observer.ts`) - Uses MutationObserver to detect DOM additions, selects elements via CSS selectors
3. **Data Extraction** (`check.ts`) - Extracts URLs/codes/names from page elements using configurable selector functions
4. **GraphQL Querying** (`request.ts`) - Sends batched GraphQL queries to user's Stash instance(s)
5. **Result Display** (`tooltip/`) - Injects checkmarks and displays tooltips with metadata

### Key Files

**`src/stashChecker.ts`** - Site-specific configuration hub
- Large switch statement on `window.location.host` for 50+ supported websites
- Each case defines CSS selectors and extraction logic for that site
- Calls `check()` with Target type (Scene/Performer/Studio/Gallery/Group/Tag) and CheckOptions

**`src/check.ts`** - Core matching logic
- `check()` function: selects elements, extracts query parameters, builds GraphQL queries
- Configurable selectors: `urlSelector`, `codeSelector`, `nameSelector`, `titleSelector`, `displaySelector`
- Handles custom display rules (color coding based on GraphQL filters)

**`src/request.ts`** - GraphQL request batching
- Batches multiple queries into single HTTP request (reduces API calls)
- Supports both GET and POST methods (configurable per endpoint)
- Manages request queue to prevent overwhelming endpoints
- Handles API key authentication

**`src/dataTypes.ts`** - Core type definitions
- `Target` enum: Scene, Performer, Gallery, Group, Studio, Tag
- `Type` enum: URL, Code, Name, Title, StashId
- `StashEndpoint`: name, url, API key
- `DataField` enum: All supported GraphQL fields

**`src/settings/`** - Configuration management
- `endpoints.ts`: Manage multiple Stash endpoints with API keys
- `storage.ts`: Browser localStorage via Greasemonkey GM API
- `menu.ts`: Tampermonkey menu integration
- `display.ts`: Custom display rules (GraphQL filter → color)
- `providers.ts`: Settings value caching and providers

**`src/tooltip/`** - Result display
- `tooltip.ts`: Injects checkmark symbols (✓/⚠/✗) into DOM
- `tooltipElement.ts`: Creates floating tooltip UI with Floating UI library
- `stashQuery.ts`: Formats tooltip content from GraphQL response

### Important Patterns

**Site Configuration Pattern:**
Each website in `stashChecker.ts` follows this pattern:
```typescript
check(Target.Scene, "CSS_SELECTOR", {
    observe: true,                        // Use MutationObserver for dynamic content
    urlSelector: e => e.closest("a")?.href,
    codeSelector: e => extractCode(e),
    displaySelector: e => e.querySelector(".title"),
    titleSelector: e => e.textContent
});
```

**Query Type Priority:**
The userscript tries query types in this order:
1. StashId (most reliable)
2. URL
3. Studio Code (for scenes)
4. Name/Title (least reliable, requires exact match)

**Batch Optimization:**
Queries are collected for 1 second before being sent as a batch to reduce API calls. Each batch can contain multiple query types (URL, Code, Name) for different targets.

## Testing

No automated test suite exists. Testing is manual:
1. Run dev server
2. Visit supported websites
3. Verify checkmarks appear correctly
4. Test tooltip display and metadata
5. Test on both light/dark themes

## Deployment

GitHub Actions automatically deploys to [Gist](https://gist.github.com/timo95/562b9363d491e3ee281cb46944445fcd) when a version tag is pushed.

## Adding Support for New Websites

To add a new website to `stashChecker.ts`:

1. Add domain to `metadata.js` `@match` array
2. Add new case in switch statement in `stashChecker.ts`
3. Determine what to check (scenes, performers, studios, etc.)
4. Inspect website HTML to find CSS selectors
5. Write selector functions to extract data (URLs, codes, names)
6. Set `observe: true` if site uses dynamic loading (SPAs)
7. Test on the website

Example:
```typescript
case "example.com": {
    check(Target.Scene, "div.video-card h3", {
        observe: true,
        urlSelector: e => e.closest("a")?.href,
        codeSelector: e => e.dataset.code,
        titleSelector: e => e.textContent?.trim()
    });
    break;
}
```

## Configuration Notes

**webpack.config.js:**
- Uses `cssimportant-loader` for files ending in `_important.scss/css` to add `!important` to all rules (needed to override website styles)
- Terser minifier configured for readability (`beautify: true`, `mangle: false`)
- Top-level await enabled
- Dev server writes to disk for userscript loading

**tsconfig.json:**
- Strict mode enabled
- Target ES2024
- DOM library included (browser environment)

## Common Troubleshooting

**"no connection" errors:**
- Check URL format: must include scheme (`http`/`https`) and end with `/graphql`
- Check API key (or leave empty if none required)
- Whitelist domain in Tampermonkey settings
- Firefox HTTPS-only mode may block `http://` URLs
- Some sites block due to CSP headers (change Tampermonkey to remove CSP header)

**GraphQL parsing errors:**
- Reverse proxies may truncate long GET URLs
- Solution: Reduce batch size in settings OR switch to POST method

**Checkmarks not appearing:**
- Open browser console and look for errors
- Verify Stash endpoint is configured correctly in settings
- Check if CSS selectors still match (websites change HTML structure)
- Verify MutationObserver is active for dynamic sites (`observe: true`)
