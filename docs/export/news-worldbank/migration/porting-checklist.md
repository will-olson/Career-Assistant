# Porting checklist

Copy one seam at a time. `manifest.json` `id` values are the boxes below. Check a box only after the new platform matches the contract file, not after the control merely exists on screen.

## Shared

- [ ] `NEWS_API_KEY` stays server-side. The browser never calls `newsapi.org`.
- [ ] News query is the company **name** (`q`), `language=en`, `sortBy=relevancy`, `pageSize=10`.
- [ ] Drop articles whose `title` is `[Removed]`, then slice to the count for that seam (5 or 3).
- [ ] Decide whether to keep the 24-hour in-memory cache. If you keep it, key it by the same name string.
- [ ] World Bank country and topic calls may stay in the browser. Catalog search stays behind your API. The indicator proxy has no screen.
- [ ] Do not port Alpha Vantage (`/api/top-stocks`, `/symbol-search`, `/financial-metrics`) as World Bank.

## `news.industry.categories` and `news.industry.category`

- [ ] Sidebar from `GET /categories` → `[{ name }]`.
- [ ] Click loads `GET /categories/{name}` → array of `{ company_name, category, news_articles: [{ title, url }] }`.
- [ ] Render an external link per article. Empty copy: "No news articles available."
- [ ] Signed-out copy if you keep the gate. The Flask route itself is not authenticated.
- [ ] Encode the category name if names can contain spaces or slashes. This app does not.

## `news.profile.favorites` and `news.profile.articles`

- [ ] `GET /favorites?user_id=` returns an **array** with `company_id` and `company_name`.
- [ ] Map `company_id` to the id you pass to the news route before fetching.
- [ ] One `GET /news/{company_id}` per favorite. Response key is `articles`, and items are full NewsAPI objects.
- [ ] Render "News for {company_name}" and a tile per `title` linking to `url`.
- [ ] Ignore extra article fields unless the new screen shows them. They are present on this route and absent on the category route.

## `news.career.client-prefetch` and `news.career.server-prompt`

- [ ] Treat the client prefetch as non-functional. Do not depend on `response.data.favorites` or `favorite.id` unless you change the API.
- [ ] Require `user_id` on `POST /career-assistant`. Load favorites in the server from your store.
- [ ] Ignore client `news_articles` and `user_favorites`, or start reading them only as an explicit behavior change.
- [ ] Fetch 3 articles per favorite name. Put `title` and `url` in the prompt as `<a href='{url}' target='_blank'>{title}</a>`.
- [ ] Keep the instruction that limits links to companies named in the question, if answers should match today.
- [ ] Preferred News Sources and Time Frame stay prompt lines unless you add real NewsAPI `sources` and `from`/`to`.
- [ ] OpenAI body: model `gpt-4o-mini`, temperature `0.7`, one user message. Return `{ response }`.
- [ ] Render `response` as HTML. `formatResponse` linkifies bare URLs and turns newlines into `<br />`.

## `wb.country.browser` and `wb.topic.browser`

- [ ] `GET https://api.worldbank.org/v2/country?format=json` and read index `1`.
- [ ] Map `id`, `name`, `region.value`, `incomeLevel.value`, `capitalCity`. Longitude and latitude are optional.
- [ ] Initial list: 4 shuffled countries. Search filters **country name**. Tiles show name, region, income level, capital.
- [ ] `GET https://api.worldbank.org/v2/topic?format=json`, map `value` → name and `sourceNote` → description.
- [ ] Empty topic query clears the list.

## `wb.catalog.data-page`

- [ ] `GET /api/search?q=` → catalog `qname=dataset`, `qterm`, `$top=10`.
- [ ] Normalize to `{ title, description, link, keywords }` from `name`, `identification.description`, `app_legacy_url`, `keywords_list`.
- [ ] Missing `q` is 400. Empty `Response.value` is `{ data: [] }`.
- [ ] Render title, HTML description, and "View Report".
- [ ] Decide what a missing URL does. Today it becomes `"#"` and still renders the anchor.
- [ ] Company search currently hides these results. Keep or drop that coupling on purpose.

## `wb.topic.career`, `wb.region.career`, `wb.catalog.career-client`, `wb.catalog.career-server`

- [ ] Topic dropdown options are `topic.value` strings. Prompt: `Focus on the topic: {selectedTopic}.`
- [ ] Region options are `country.region.value` with duplicates. Prompt: `Focus on the region: {selectedRegion}.`
- [ ] Do not call `GET /api/world-bank` for these dropdowns.
- [ ] Report typeahead uses `GET /api/search`. Skip the mount call that omits `q`.
- [ ] Client filter: normalized substring. Server filter: exact trimmed lowercase title.
- [ ] Replace the `http://localhost:5555/api/search` self-call with an in-process call, and encode `q`.
- [ ] On a match, append title, description, and link to the prompt. On a miss, append `No related report found.`

## `wb.indicator.unused`

- [ ] Leave it unwired if the new UI only recreates current screens.
- [ ] If you add a chart, treat paging, `date`, and a `value` field map as new work. This handler returns the raw envelope and defaults to `SP.POP.TOTL`.

## After the port

- [ ] Industry News: signed out message, then a category, then title links that open in a new tab.
- [ ] Profile: a favorite shows "News for {name}" and the same title-link treatment.
- [ ] Data: four country tiles, a topic search hit with a description, a report search hit with "View Report".
- [ ] Career Assistant: a signed-in question returns `{ response }` whose news links come from server-side favorites, not from the client `news_articles` array.
- [ ] Confirm Preferred News Sources still do not change the NewsAPI request, unless that was an intentional change.
- [ ] Confirm a selected catalog report survives without Flask listening on `localhost:5555`.
