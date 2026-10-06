# Trace: Data page catalog search

“Search reports” on `/data`. Flask proxies the World Bank Data Catalog and returns four fields.

## 1. User action and route

Route: `/data`, signed in, company search box empty.

The user types in the input whose placeholder is “Search reports”. Each change with a non-empty trimmed query calls `/api/search`. Clearing the box sets `catalogResults` to `[]`.

Results render under the four hint tiles: title, HTML description, and “View Report” when `link` is truthy. There is no debounce in code. Each `onChange` fires a request.

If the user has also typed in “Search companies”, the whole block that contains catalog results is unmounted (`userId && !searchQuery`), even if `catalogResults` is still in state.

## 2. Client call

```176:215:client/src/components/DataPage.js
    const [dataCatalogQuery, setDataCatalogQuery] = useState('');
    const [catalogResults, setCatalogResults] = useState([]);

    const handleDataCatalogSearchChange = (event) => {
        const query = event.target.value.trim();
        setDataCatalogQuery(query);
        if (query.length > 0) {
            fetchCatalogResults(query);
        } else {
            setCatalogResults([]);
        }
    };

    const fetchCatalogResults = async (query) => {
        try {
            const response = await axios.get('http://localhost:5555/api/search', {
                params: { q: query }
            });
    
            console.log("Full API Response:", response);
    
            console.log("Response Data Field:", response.data);
    
            if (Array.isArray(response.data.data)) {
                console.log("'data' is an array with length:", response.data.data.length);
            } else {
                console.warn("'data' is not an array or missing");
            }
    
            if (Array.isArray(response.data.data) && response.data.data.length > 0) {
                console.log("Setting catalog results with data:", response.data.data);
                setCatalogResults(response.data.data);
            } else {
                console.log("No results found, resetting catalog results.");
                setCatalogResults([]);
            }
        } catch (err) {
            console.error('Error fetching data catalog results:', err);
        }
    };
```

The live function also logs the raw Axios response before the `if`. Those logs do not affect the list.

```http
GET http://localhost:5555/api/search?q={query}
```

Host spelling is `localhost`, not `127.0.0.1`.

## 3. Flask handler and upstream HTTP

```303:363:server/app.py
@app.route('/api/search', methods=['GET'])
def search_catalog():
    query = request.args.get('q')
    if not query:
        return jsonify({'error': 'Query parameter "q" is required'}), 400

    url = 'https://datacatalogapi.worldbank.org/ddhxext/Search'
    params = {
        'qname': 'dataset',
        'qterm': query,
        '$top': 10
    }

    try:
        
        response = requests.get(url, params=params)

        
        print("Response Status Code:", response.status_code)
        print("Raw Response Text:", response.text)

        response.raise_for_status()
        external_data = response.json()

        
        print("External API Response (JSON):", external_data)

        
        items = external_data.get("Response", {}).get("value", [])
        if not isinstance(items, list) or not items:
            print("No valid items found in 'value' key.")
            return jsonify({'data': []})

        
        processed_data = []
        for item in items:
            title = item.get("name", "No Title")
            description = item.get("identification", {}).get("description", "No Description")
            link = item.get("app_legacy_url", "#")
            keywords = item.get("keywords_list", [])

            
            processed_data.append({
                "title": title,
                "description": description,
                "link": link,
                "keywords": keywords
            })

        
        print("Processed Data:", processed_data)

        
        return jsonify({'data': processed_data})

    except requests.exceptions.RequestException as e:
        print("Error fetching data:", e)
        return jsonify({'error': 'Failed to fetch data from external API'}), 500
    except Exception as e:
        print("Unexpected error:", e)
        return jsonify({'error': 'An unexpected error occurred'}), 500
```

```http
GET https://datacatalogapi.worldbank.org/ddhxext/Search?qname=dataset&qterm={q}&$top=10
```

`description` is often HTML. `link` falls back to the string `"#"`, which is truthy in React.

## 4. Response contract

[../contracts/datacatalog.search.json](../contracts/datacatalog.search.json)

## 5. JSX

```257:265:client/src/components/DataPage.js
                        <div className="topic-search">
                            <input
                                type="text"
                                placeholder="Search reports"
                                value={dataCatalogQuery}
                                onChange={handleDataCatalogSearchChange}
                                className="search-input"
                            />
                        </div>
```

```280:315:client/src/components/DataPage.js
            {userId && !searchQuery && (
                <>
                    <div className="tiles-container">
                        <div className="tile">
                            <em>Use company search to explore publicly traded companies and financial metrics.</em>
                        </div>
                        <div className="tile">
                            <em>Review topics to guide key considerations for global analyses.</em>
                        </div>
                        <div className="tile">
                            <em>Use region search to view income levels and capital cities.</em>
                        </div>
                        <div className="tile">
                            <em>Access data and reporting from The World Bank.</em>

                        </div>
                    </div>

                    <div className="data-catalog-results">
                        {catalogResults.length > 0 && (
                            catalogResults.map((item, index) => (
                            <div key={index} className="data-catalog-item">
                                <h4>{item.title}</h4>
                                <p 
                                className="catalog-description" 
                                dangerouslySetInnerHTML={{ __html: item.description }}
                                ></p>
                                {item.link && (
                                <a href={item.link} target="_blank" rel="noopener noreferrer">
                                    View Report
                                </a>
                                )}
                            </div>
                            ))
                        )}
                        </div>
```

`keywords` is on the JSON and is not rendered.

## 6. Reproduce vs drop

Reproduce:

- A search box that queries a catalog proxy with `q`.
- At most 10 datasets.
- Title, HTML description, and a report link.
- Hide or keep results when another search is active. Today, company search hides them.

Replace on a new platform:

- `dangerouslySetInnerHTML` on catalog HTML. Sanitize or render as text unless you trust the catalog payload.
- The `"#"` default. Treat a missing `app_legacy_url` as no link, or you will show “View Report” pointing at `#`.
- Per-keystroke requests. Add a debounce if the new host rate-limits.

Drop:

- `keywords`, unless the new UI shows them.
- The Flask `print` of the raw upstream body. It is debug output, and it can be large.
