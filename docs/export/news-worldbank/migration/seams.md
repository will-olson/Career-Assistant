# Seams: live HTTP vs prompt text

Use this when deciding what a new platform must call, and what it only has to mention to a model.

## Live HTTP

These calls leave the process and return data the UI or the prompt then reads.

| Seam | Who calls | Upstream | What comes back |
|------|-----------|----------|-----------------|
| `newsapi.fetch` | Flask only | `GET https://newsapi.org/v2/everything` | Up to 10 English articles, relevancy sort, query is the company name |
| `news.industry.category` | `IndustryNews` → Flask | NewsAPI via the choke point, count 5 | `{ company_name, category, news_articles: [{ title, url }] }` |
| `news.profile.articles` | `ProfilePage` → Flask | NewsAPI via the choke point, count 5 | `{ company_name, articles: [full article] }` |
| `news.career.server-prompt` | Flask during `POST /career-assistant` | NewsAPI via the choke point, count 3 per favorite | Title and URL only, wrapped in HTML anchors inside the prompt |
| `wb.country.browser` | `DataPage` in the browser | `GET https://api.worldbank.org/v2/country?format=json` | Envelope `[metadata, countries]`. UI uses name, `region.value`, `incomeLevel.value`, `capitalCity` |
| `wb.topic.browser` | `DataPage` in the browser | `GET https://api.worldbank.org/v2/topic?format=json` | Envelope `[metadata, topics]`. UI uses `value` and `sourceNote` |
| `wb.topic.career` | `CareerAssistant` in the browser | Same topic URL | Name strings for a dropdown |
| `wb.region.career` | `CareerAssistant` in the browser | Same country URL | `region.value` for every country, duplicates kept |
| `wb.catalog.data-page` | `DataPage` → Flask | `GET https://datacatalogapi.worldbank.org/ddhxext/Search` | `{ title, description, link, keywords }`, max 10 |
| `wb.catalog.career-client` | `CareerAssistant` → Flask | Same catalog URL | Same object. Client filters titles locally |
| `wb.catalog.career-server` | Flask → `http://localhost:5555/api/search` | Same catalog URL, second hop | Title, description, and link pasted into the prompt when the title matches exactly |
| `wb.indicator.unused` | Nobody in the UI | `GET http://api.worldbank.org/v2/country/all/indicator/{id}` | Raw envelope. Default id `SP.POP.TOTL` |

Favorites are local database reads, not NewsAPI or World Bank. They are the input list for profile news and for the Career Assistant news loop.

```http
GET /favorites?user_id={id}
```

Body is an array of `{ company_id, company_name, link, indeed, category }`.

## Prompt text only

`POST /career-assistant` copies these body fields into the OpenAI user message. They do not change NewsAPI query params or World Bank indicator requests.

| Body field | Prompt line | UI control |
|------------|-------------|------------|
| `preferred_sources` | `Preferred News Sources: Reuters, ...` | Checkboxes |
| `time_frame` | `Time Frame: Last 30 days` | Radios: Last 7 days, Last 30 days, Last 6 months |
| `selectedTopic` | `Focus on the topic: {name}.` | Topic dropdown |
| `selectedRegion` | `Focus on the region: {name}.` | Region dropdown |
| `selectedCompany` | `Consider the company: {name}.` | Company dropdown from `GET /companies` |
| `scope_of_analysis` | `Scope of Analysis: ...` | Checkboxes |
| `sentiment_tone` | `Sentiment Tone: ...` | Select |
| `level_of_detail` | `Level of Detail: ...` | Select |
| `industry_focus` | `Industry Focus: ...` | Checkboxes |
| `specific_topics` | `Specific Topics: ...` | Text |
| `preferred_format` | `Preferred Format: ...` | Radios |

`time_frame` is also a dependency of the client news `useEffect`. That effect does not call NewsAPI with a date. With the current favorites bug it does not call NewsAPI at all. See [known-breaks.md](known-breaks.md).

## In the prompt, but from a live call

| Source | How it enters the prompt |
|--------|--------------------------|
| Favorite companies | DB query by `user_id`. Lines: `- {name} (Category: {category})` |
| NewsAPI | Three articles per favorite: `  - <a href='{url}' target='_blank'>{title}</a>` |
| Data Catalog | Only if `selectedReport` is non-empty and a returned `title` equals it after strip and lowercasing |

The model is instructed to include news links only for companies named in the user's question, even though the prompt contains headlines for every favorite.

## Ignored on the server

The client POST includes `user_favorites` and `news_articles`. `career_assistant` never reads them. Personalization is `user_id` plus the database.

## OpenAI call

```http
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer {OPENAI_API_KEY}
```

```json
{
  "model": "gpt-4o-mini",
  "messages": [{ "role": "user", "content": "<assembled prompt>" }],
  "temperature": 0.7
}
```

The Flask response is `{ "response": "<assistant text>" }`.

## What users see, by surface

| Surface | Live data on screen | Model synthesis |
|---------|---------------------|-----------------|
| `/industry-news` | Title links, 5 per company | No |
| `/profile` | Title links, 5 per favorite | No |
| `/data` | Country tiles, topic blurbs, catalog HTML | No |
| `/career-assistant` | Topic names, region labels, catalog title and HTML while picking a report | Yes. News and the chosen report are context, not tables |
