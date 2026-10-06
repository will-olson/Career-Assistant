# Known breaks

Port these as behavior, or fix them on purpose. A line-by-line copy that "corrects" one of them will not match what this app does.

## 1. Career Assistant reads the wrong favorites shape

`GET /favorites` returns a JSON array:

```json
[{ "company_id": 1, "company_name": "Stripe", "link": "...", "indeed": "...", "category": "Fintech" }]
```

`CareerAssistant.fetchUserFavorites` continues only when `response.data.favorites` is set. For an array, that is undefined, so `userFavorites` becomes `[]`.

`fetchNewsArticles` then asks for `favorite.id` and `favorite.name`. Those keys are not on the favorites payload. `GET /profile` does return `{ favorites: [{ id, company_name, ... }] }`, and no screen in this package calls it.

Profile works because `ProfilePage` maps `company_id` → `id` and `company_name` → `name` before `GET /news/{id}`.

Effect: the Career Assistant client prefetch is dead. Answers still include favorite headlines because the server loads favorites by `user_id`.

The POST also appends `Please check if any of the following companies are mentioned: ${companyNames}`. `companyNames` is empty, so that sentence names nobody. The server prompt still lists the real favorites.

## 2. Category news and company news are different shapes

| Route | Article fields |
|-------|----------------|
| `GET /categories/<name>` | `title`, `url` only |
| `GET /news/<id>` | NewsAPI article object after the `[Removed]` filter and a slice of 5 |

Profile renders `title` and `url` from the full object. A client written against the category payload will ignore extra fields. A client written against `/news` and pointed at the category route will not find `articles`; the key there is `news_articles`.

## 3. Preferred sources and time frame do not filter NewsAPI

Checkboxes and radios are prompt lines. `fetch_news_for_company` always sends `language=en`, `sortBy=relevancy`, `pageSize=10`, and `q={company name}`.

There is no `from`, `to`, or `sources` parameter. Cache TTL is 86400 seconds, so a headline can be up to a day old in `news_cache` even when the user picks "Last 7 days".

## 4. Report search on Career Assistant mount

```javascript
useEffect(() => {
  fetchTopics();
  fetchCompanies();
  fetchRegions();
  fetchReports();
}, []);
```

`fetchReports()` with no argument calls `GET /api/search` without `q`. The handler returns 400:

```json
{ "error": "Query parameter \"q\" is required" }
```

Typing in "Choose a Report" is what populates `reports`.

## 5. Two report filters

The client keeps a row when the normalized title contains the normalized input (`[^a-z0-9]` removed, lowercased).

The server, on submit, keeps a row only when `title.strip().lower()` equals `selectedReport.strip().lower()`.

A partial string can show a row in the dropdown and still produce `No related report found.` in the prompt. Clicking a row sets the input to the full title, which satisfies the server match if that title is still in the top 10 for `q={full title}`.

## 6. Catalog link default is `#`

`app_legacy_url` missing becomes `"#"`. Both UIs treat any truthy `link` as a real anchor, so "View Report" and "Read full report" can point at `#`.

`description` is inserted with `dangerouslySetInnerHTML` on the Data page and in the report picker, and the raw HTML is also concatenated into the GPT prompt.

## 7. Region dropdown is not a region list

`fetchRegions` maps every country to `country.region.value` and does not unique the array. "North America" appears once per country in that region.

The Data page control labeled "Search regions" filters `country.name`, not `region`.

First paint is `sort(() => Math.random() - 0.5).slice(0, 4)`. Clearing the box uses the same shuffle and `slice(0, 5)`.

## 8. Catalog results hide when company search is non-empty

On `/data`, catalog HTML is inside `{userId && !searchQuery && (...)}`. A company-symbol query unmounts report results, country tiles, and the hint tiles together. Topic results sit outside that condition and stay visible.

## 9. Career Assistant catalog lookup calls localhost

```python
response = requests.get(f"http://localhost:5555/api/search?q={selected_report}")
```

The port and host are fixed. `selected_report` is not query-encoded. Moving the API off port 5555, or off the same machine, makes every selected report resolve as `Failed to retrieve report data.`

## 10. Host spellings

| Caller | Base |
|--------|------|
| Industry News, Data page companies | `http://127.0.0.1:5555` |
| Career Assistant favorites, news, search, POST; Data page catalog | `http://localhost:5555` |
| Profile favorites and news | relative, CRA proxy to `http://localhost:5555` |

`127.0.0.1` and `localhost` are the same Flask app in local dev. A deployed origin that only allows one of them will break the other callers. Relative Profile calls fail in any build that does not proxy `/favorites` and `/news` to Flask.

## 11. Auth is localStorage, not the session

`POST /login` returns `user_id` and does not set `session['user_id']`. `POST /logout` pops that session key anyway. Industry News, Data, and Career Assistant gate on `localStorage.user_id`. The news and catalog routes do not check it. Anyone who can reach Flask can call `GET /categories/{name}`, `GET /news/{id}`, and `GET /api/search` without a user.

`POST /career-assistant` checks that `user_id` is present in JSON. It does not check a session or a password on that request.

## 12. Indicator route is unused

`GET /api/world-bank` defaults to `SP.POP.TOTL` and returns the raw World Bank array. No component fetches it. Wiring it to the Data page is new product behavior. See [../traces/08-indicator-unused.md](../traces/08-indicator-unused.md).

## 13. Cache key is the company name

`news_cache` is a process dict. Restart clears it. `"Stripe"` and `"stripe"` miss each other. Industry News and Profile share entries when the name string matches, which is why the second screen can avoid a NewsAPI call for 24 hours.

## 14. Client prompt sentence vs stored question

`handleSubmit` replaces `prompt` in the POST body with the textarea plus the "companies are mentioned" sentence. The chat stores `question: inputs.prompt`, the textarea value, not that expanded string. The model sees the expanded string. The user sees the textarea text as `Q:`.
