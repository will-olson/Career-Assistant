# Trace: NewsAPI choke point

Shared server function. No screen calls NewsAPI directly.

## 1. User action and route

None. Industry News, Profile, and Career Assistant reach this function through Flask.

| Caller | Route | `desired_article_count` |
|--------|-------|-------------------------|
| `view_companies_in_category_and_news` | `GET /categories/<category_name>` | 5 (default) |
| `get_news_for_company` | `GET /news/<int:company_id>` | 5 (default) |
| `career_assistant` | `POST /career-assistant` | 3 |

## 2. Client call

There is no client call to `newsapi.org`.

## 3. Flask handler and upstream HTTP

Cache constants:

```22:23:server/app.py
news_cache = {}
CACHE_TTL = 86400
```

Choke point:

```36:57:server/app.py
def fetch_news_for_company(company_name, desired_article_count=5):
    current_time = time.time()
    if company_name in news_cache:
        cached_data, timestamp = news_cache[company_name]
        if current_time - timestamp < CACHE_TTL:
            return [article for article in cached_data if article['title'] != '[Removed]'][:desired_article_count]


    response = requests.get("https://newsapi.org/v2/everything", params={
        "q": company_name,
        "apiKey": NEWS_API_KEY,
        "language": "en",
        "sortBy": "relevancy",
        "pageSize": 10
    })

    if response.status_code == 200:
        articles = response.json().get('articles', [])
        news_cache[company_name] = (articles, current_time)
        return [article for article in articles if article['title'] != '[Removed]'][:desired_article_count]
    else:
        return []
```

`NEWS_API_KEY = os.getenv("NEWS_API_KEY")` at line 18. The cache stores the raw `articles` list. The `[Removed]` filter and the slice run on both the cache hit and the fresh response. A non-200 returns `[]` and does not write the cache. The key is the company name string, so `"Stripe"` and `"stripe"` are different entries.

Query that is actually sent:

```http
GET https://newsapi.org/v2/everything?q={company_name}&apiKey={NEWS_API_KEY}&language=en&sortBy=relevancy&pageSize=10
```

Not sent: `from`, `to`, `sources`, `domains`, `excludeDomains`.

## 4. Response contract

- Request: [../contracts/newsapi.request.json](../contracts/newsapi.request.json)
- Article fields this app reads: [../contracts/newsapi.article.json](../contracts/newsapi.article.json)

Return value is a Python list of article dicts, length at most `desired_article_count`.

## 5. JSX

None in this trace. Rendering is in the Industry News, Profile, and Career Assistant traces.

## 6. Reproduce vs drop

Reproduce:

- One function that takes a company **name** and a count.
- The query above, a 10-article fetch, a `[Removed]` title drop, then a slice.
- A 24-hour cache keyed by that name if you need the same freshness behavior. The cache is process memory and dies on restart.

Drop or replace when the target platform has its own cache:

- The module-level `news_cache` dict.
- Logging around callers. It is not part of the contract.

Do not invent date or source filters here. Those UI controls are documented in the Career Assistant trace and do not reach this function.
