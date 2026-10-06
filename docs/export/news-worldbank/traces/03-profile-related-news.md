# Trace: Profile related news

Personalized headlines for favorited companies. One news request per favorite.

## 1. User action and route

Routes that mount `ProfilePage`: `/profile`, and `/` after login (`client/src/components/App.js`).

The page loads favorites for `loggedInUser.id`, remaps them, then requests news for each company. “Related News” is a section per company. Each article is a tile whose text is the title and whose href is the article URL.

Empty favorites: “No favorite companies yet.” Empty news array: “No news available for your favorited companies.” A company with zero articles: “No news available.” (`News.js`).

Signed out: “Please log in to view your profile.”

## 2. Client call

Favorites, then remap. This remap is why Profile news works and the Career Assistant prefetch does not.

```33:57:client/src/components/ProfilePage.js
  const fetchFavorites = async (userId) => {
    try {
      const response = await fetch(`/favorites?user_id=${userId}`, {
        method: 'GET',
        headers: {
          'Content-Type': 'application/json',
        },
        credentials: 'include',
      });

      if (response.ok) {
        const favoritesData = await response.json();

        
        const updatedFavorites = favoritesData.map(favorite => ({
          id: favorite.company_id,
          name: favorite.company_name,
          link: favorite.link,
          category: favorite.category,
          indeed: favorite.indeed,
        }));

        setFavorites(updatedFavorites);
        fetchNewsForFavorites(updatedFavorites);
      }
```

```64:75:client/src/components/ProfilePage.js
  const fetchNewsForFavorites = async (favoriteCompanies) => {
    const newsPromises = favoriteCompanies.map(async (favorite) => {
      const response = await fetch(`/news/${favorite.id}`);
      if (response.ok) {
        const newsData = await response.json();
        return {
          company_name: newsData.company_name,
          articles: newsData.articles
        };
      }
      return { company_name: favorite.name, articles: [] };
    });
```

Transport is the CRA proxy (`client/package.json` `"proxy": "http://localhost:5555"`), because the paths are relative.

```http
GET /favorites?user_id={userId}
GET /news/{company_id}
```

N favorites produce N news requests. The server cache makes later requests for the same company name cheap for 24 hours, including requests that came from Industry News first.

`GET /profile` returns `{ favorites: [{ id, company_name, ... }] }` and is not called here. Contract for that unused shape: [../contracts/profile.get.response.json](../contracts/profile.get.response.json).

## 3. Flask handler and upstream HTTP

```460:468:server/app.py
    if request.method == 'GET':
        favorites = Favorites.query.filter_by(user_id=user_id).all()
        return jsonify([{
            "company_id": fav.company_id,
            "company_name": fav.company.name,
            "link": fav.company.link,
            "indeed": fav.company.indeed,
            "category": fav.company.category.name if fav.company.category else None
        } for fav in favorites]), 200
```

```518:532:server/app.py
@app.route('/news/<int:company_id>', methods=['GET'])
def get_news_for_company(company_id):

    company = Company.query.get(company_id)
    if not company:
        return jsonify({"message": "Company not found"}), 404


    news_articles = fetch_news_for_company(company.name)


    return jsonify({
        "company_name": company.name,
        "articles": news_articles
    })
```

`articles` is the full NewsAPI object list (after filter and slice of 5), not the `{title, url}` strip used by the category route.

Upstream: [01-newsapi-choke-point.md](01-newsapi-choke-point.md).

## 4. Response contract

- Favorites: [../contracts/favorites.get.response.json](../contracts/favorites.get.response.json)
- News: [../contracts/company-news.response.json](../contracts/company-news.response.json)

## 5. JSX

Section shell:

```146:157:client/src/components/ProfilePage.js
      <div className="content-square">
        <h2 className="section-title">Related News</h2>
        <div className="news-tile">
          {news.length > 0 ? (
            news.map((newsItem, index) => (
              <News key={index} news={newsItem} />
            ))
          ) : (
            <p>No news available for your favorited companies.</p>
          )}
        </div>
      </div>
```

Tile grid. Only `title` and `url` are read:

```3:22:client/src/components/News.js
function News({ news }) {
    return (
        <div className="container">
            <h2 className="section-title">News for {news.company_name}</h2>
            {news.articles.length === 0 ? (
                <p>No news available.</p>
            ) : (
                <div className="company-tiles">
                    {news.articles.map((article, index) => (
                        <div key={index} className="content-square">
                            <a href={article.url} className="link" target="_blank" rel="noopener noreferrer">
                                {article.title}
                            </a>
                        </div>
                    ))}
                </div>
            )}
        </div>
    );
}
```

## 6. Reproduce vs drop

Reproduce:

- Favorites list, then one news call per company id.
- Map `company_id` to the id you send to the news route. If you keep the current API, that field is `company_id`, not `id`.
- A heading “News for {company}” and linked titles.
- Accept that the JSON may contain more article fields than the UI shows. If the new screen should show dates or images, they are already on this route and absent on the category route.

Drop:

- `credentials: 'include'` unless the new host uses cookies. Login state for this page is `loggedInUser`, which `App` sets from `localStorage`.
- The unused `GET /profile` wrapper, unless you intentionally switch the client to that shape (`id` instead of `company_id`).
