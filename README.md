# SlideHub MCP Server for Claude

Connect [Claude](https://claude.ai) to your [SlideHub](https://slidehub.com) slide library. Search, generate, and manage professional presentations — directly from Claude's desktop and web apps.

## Prerequisites

- A [Claude](https://claude.ai) Pro, Team, or Enterprise account
- **A paid [SlideHub](https://slidehub.com) account with MCP access enabled** — this integration requires an active SlideHub subscription. To learn more or get started, visit [slidehub.com](https://slidehub.com) or contact [success@slidehub.com](mailto:success@slidehub.com)

## Setup

### 1. Add the MCP server in Claude

1. Open [Claude](https://claude.ai) (desktop app or web)
2. Go to **Settings** > **Connectors** (or **MCP Servers**)
3. Add a new MCP server with the URL:
   ```
   https://ppt.slidehub.io/api/mcp/slidehub
   ```

### 2. Authenticate

On first use, Claude will open a browser window to sign in with your SlideHub account. No API keys required.

### 3. Verify the connection

Ask Claude:

```
Can you validate my SlideHub access?
```

## What you can do

### Search your slide library

Find slides using keyword search or AI-powered semantic search across your company's entire library.

```
Search for slides about quarterly revenue
```

### Generate slides from text

Turn any text content into a professionally designed slide using your company's templates and brand.

```
Generate a slide from this text: "Q1 revenue grew 23% YoY driven by enterprise expansion"
```

### Fill placeholders

Populate dynamic slides with specific values — names, dates, figures — and download the result.

```
Find slides with placeholders for our monthly report
```

### Combine slides into a presentation

Merge multiple slides into a single downloadable PPTX file.

```
Combine these three slides into one presentation
```

### Browse key presentations

Access your company's featured and quick-access presentations.

```
Show me the key presentations
```

## Available tools

| Tool | Description |
|------|-------------|
| `validate_access` | Verify connection and get user/company info |
| `get_team_information` | List available teams |
| `get_base_label_information` | Discover categories, subcategories, and tags |
| `get_key_presentations` | Fetch featured presentations |
| `get_slide_libraries` | Get available slide, icon, and image libraries |
| `search_assets` | Search slides by keyword with filters |
| `ai_chat_asset_search` | AI-powered semantic search across slides |
| `get_asset_details` | Get detailed info and thumbnails for specific slides |
| `get_slide_preview` | Get a visual thumbnail preview of a slide |
| `get_placeholders` | Check slides for fillable dynamic content |
| `fill_placeholders` | Fill placeholder values and generate a presentation |
| `validate_text_input` | Validate text before slide generation |
| `evaluate_content` | Optimize text content for slide generation |
| `find_best_inspiration` | Find the best template for given content |
| `generate_slide` | Generate a slide from text using a template |
| `check_job_status` | Poll the status of a generation job |
| `get_slide_result` | Render completed results with previews |
| `download_presentation` | Get a download URL for a generated presentation |
| `quick_generate` | Generate a slide in a single call |
| `combine_slides` | Merge multiple slides into one PPTX |

## Support

- Documentation: [help.slidehub.io](https://help.slidehub.io)
- Contact: [success@slidehub.com](mailto:success@slidehub.com)
- Website: [slidehub.com](https://slidehub.com)

## License

Proprietary. Copyright SlideHub.
