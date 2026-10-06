# Trace: Industry News

Browse-by-industry headline list. Title and external link only.

## 1. User action and route

Route: `/industry-news` (`client/src/components/App.js` registers the route).

Nav always shows “Industry News” (`client/src/components/NavBar.js`). The page itself checks `localStorage` `user_id`.

Signed out, the only content is one tile: “You must be logged in to view industry news and categories.”

Signed in:

1. On mount, `GET /categories` fills the sidebar.
2. Click a category name. `GET /categories/{name}` returns companies and headlines.
3. Each company is a heading plus a list of links. Empty list copy: “No news articles available.” No category selected: “Please select a category to view companies and news articles.” No companies: “No companies found for this category.”

There are no images, dates, or snippets.

## 2. Client call

```9:35:client/src/components/IndustryNews.js
    useEffect(() => {
        const fetchCategories = async () => {
            try {
                const response = await fetch('http://127.0.0.1:5555/categories');
                if (response.ok) {
                    const data = await response.json();
                    setCategories(data);
                }
            } catch (error) {
                console.error('Error fetching categories:', error);
            }
        };

        fetchCategories();

        
        const handleStorageChange = () => {
            setUserId(localStorage.getItem('user_id'));
        };

        window.addEventListener('storage', handleStorageChange);

        
        return () => {
            window.removeEventListener('storage', handleStorageChange);
        };
    }, []);
```

```37:49:client/src/components/IndustryNews.js
    const handleCategoryClick = async (categoryName) => {
        setSelectedCategory(categoryName);

        try {
            const response = await fetch(`http://127.0.0.1:5555/categories/${categoryName}`);
            if (response.ok) {
                const data = await response.json();
                setCompanies(data);
            }
        } catch (error) {
            console.error('Error fetching companies and news:', error);
        }
    };
```

Host is `127.0.0.1:5555`, not the CRA proxy. Category names go into the path unencoded.

```http
GET http://127.0.0.1:5555/categories
GET http://127.0.0.1:5555/categories/{categoryName}
```

Sidebar response is `[{ "name": "Fintech" }, ...]`.

## 3. Flask handler and upstream HTTP

```495:516:server/app.py
@app.route('/categories', methods=['GET'])
def get_categories():
    categories = Category.query.all()
    return jsonify([{"name": category.name} for category in categories])

@app.route('/categories/<category_name>', methods=['GET'])
def view_companies_in_category_and_news(category_name):
    category = Category.query.filter_by(name=category_name).first()
    if not category:
        return jsonify({"message": "Category not found."}), 404

    companies = Company.query.filter_by(category_id=category.id).all()
    companies_info = []
    for company in companies:
        news_articles = fetch_news_for_company(company.name)
        companies_info.append({
            "company_name": company.name,
            "category": category.name,
            "news_articles": [{"title": article['title'], "url": article['url']} for article in news_articles]
        })

    return jsonify(companies_info)
```

Each company name calls [01-newsapi-choke-point.md](01-newsapi-choke-point.md) with the default count of 5. The handler then keeps only `title` and `url`.

## 4. Response contract

[../contracts/category-news.response.json](../contracts/category-news.response.json)

## 5. JSX

```76:110:client/src/components/IndustryNews.js
                    <main className="news-content">
                        {selectedCategory ? (
                            <>
                                <h2>{selectedCategory} Companies and News</h2>
                                {companies.length > 0 ? (
                                    companies.map((company) => (
                                        <div key={company.company_name} className="company-section">
                                            <h3>{company.company_name}</h3>
                                            <ul>
                                                {company.news_articles.length > 0 ? (
                                                    company.news_articles.map((article, index) => (
                                                        <li key={index}>
                                                            <a
                                                                href={article.url}
                                                                target="_blank"
                                                                rel="noopener noreferrer"
                                                            >
                                                                {article.title}
                                                            </a>
                                                        </li>
                                                    ))
                                                ) : (
                                                    <li>No news articles available.</li>
                                                )}
                                            </ul>
                                        </div>
                                    ))
                                ) : (
                                    <p>No companies found for this category.</p>
                                )}
                            </>
                        ) : (
                            <p>Please select a category to view companies and news articles.</p>
                        )}
                    </main>
```

Login gate:

```52:56:client/src/components/IndustryNews.js
        <div className="industry-news-container">
            {!userId ? (
                <div className="company-tile">
                    <h2>You must be logged in to view industry news and categories.</h2>
                </div>
```

## 6. Reproduce vs drop

Reproduce:

- Sidebar of category names, then one request that returns companies plus up to five `{title, url}` links.
- External links with `target="_blank"`.
- The three empty strings above.
- A signed-in gate if the product still treats industry news as private. The API itself does not check `user_id`.

Drop:

- Per-article description, image, and date. This screen never had them because the server strips the payload.
- Any NewsAPI source or date parameter. This path does not send them.
