# AI/ML Crawler Blocklist — robots.txt

Complete list of crawler UAs to block in `public/robots.txt`. Compiled June 21, 2026. Update this list as new crawlers emerge.

## Block All (User-agent: *)

```
Disallow: /api/
Disallow: /admin/
Disallow: /dashboard/
Disallow: /partners/
Disallow: /command-center/
Disallow: /trading-dashboard/
Disallow: /get-started/
Disallow: /setup-admin/
```

## Named AI/ML Crawlers (individual User-agent entries)

| Crawler | Source | Notes |
|---------|--------|-------|
| GPTBot | OpenAI | Main crawler |
| ChatGPT-User | OpenAI | User-facing ChatGPT |
| Claude-Web | Anthropic | Web crawler |
| Claude | Anthropic | Main bot |
| anthropic-ai | Anthropic | Alternative UA |
| Google-Extended | Google | Google's AI training crawler |
| GoogleOther | Google | Fallback Google crawler |
| Bytespider | ByteDance/TikTok | TikTok data scraping |
| CCBot | Common Crawl | Academic/research crawler |
| Amazonbot | Amazon | AWS-linked crawler |
| OAI-SearchBot | OpenAI Search | Search product |
| DeepSeek | DeepSeek | Chinese LLM crawler |
| Ai2bot | AI2/Mosaic | Academic research |
| Diffbot | Diffbot | Commercial data extraction |
| FacebookBot | Meta | Social media crawler |
| Applebot | Apple | Siri/Safari crawler |
| DuckDuckBot | DuckDuckGo | Search |
| Yandex | Yandex | Russian search |
| BingBot | Microsoft | Search (also in *) |
| MJ12Bot | Majestic | SEO/backlink crawler |
| AhrefsBot | Ahrefs | SEO tool |
| SemrushBot | SEMrush | SEO tool |
|Omgili | Omvili | Web archive |
|OmgiliBot | Omvili | Web archive |
| GPTBot (duplicate) | OpenAI | Listed twice in production robots.txt — cleanup pending |

## Pattern-Based Blocks (Generic)

```
User-agent: *
Crawl-delay: 1
Disallow: /*?
Disallow: /*.json$
Disallow: /*.xml$
```

## Maintenance

Check every 3 months for new crawlers. Key sources:
- OpenAI's robots.txt (ai.com, openai.com)
- Anthropic's robots.txt
- Google AI summaries (search for "Google-Extended" updates)
- [RobotRules.org](https://www.robotrules.org/) database

When adding a new crawler, use `Disallow: /` to block the entire site.
