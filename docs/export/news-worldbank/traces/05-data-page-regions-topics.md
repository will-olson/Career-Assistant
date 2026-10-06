# Trace: Data page regions and topics

Browser calls the World Bank country and topic APIs. Flask is not on this path.

The same page also renders Alpha Vantage stock tiles. Those calls are `GET /api/top-stocks`, `GET /symbol-search`, and `GET /financial-metrics/:symbol`. Leave them out of a World Bank port.

## 1. User action and route

Route: `/data`.

Signed-out copy: “You must be logged in to view data insights.” The country fetch still runs on mount, before that message matters, because `fetchTopCountries()` is in the same `useEffect` as the companies fetch and is not inside the `userId` check. The tiles that show the result are inside `{userId && !searchQuery && (...)}`, so a signed-out user does not see them.

Signed in, with the company search box empty:

- Hint tile: “Use region search to view income levels and capital cities.”
- Another hint: “Review topics to guide key considerations for global analyses.”
- “Search regions” filters **country names** and replaces the country tiles.
- “Search topics” shows topic name and description above the hint tiles. Clearing the topic box clears the list.
- On first load the country area is four random countries, not a curated region list. Labels say Region, Income Level, and Capital City. Longitude and latitude are mapped and not rendered.

“Search regions” is the country endpoint. There is no `/v2/region` call.

## 2. Client call

Initial four countries:

```38:55:client/src/components/DataPage.js
    const fetchTopCountries = async () => {
        try {
            const countriesResponse = await axios.get('https://api.worldbank.org/v2/country?format=json');
            const countries = countriesResponse.data[1].map(country => ({
                id: country.id,
                name: country.name,
                region: country.region.value,
                incomeLevel: country.incomeLevel.value,
                capitalCity: country.capitalCity,
                longitude: country.longitude,
                latitude: country.latitude,
            }));
            const shuffledCountries = countries.sort(() => Math.random() - 0.5);
            setTopCountries(shuffledCountries.slice(0, 4));
        } catch (err) {
            console.error('Error fetching top countries:', err);
        }
    };
```

Search. A non-empty query filters `country.name`. An empty query shuffles and keeps five. The placeholder says “Search regions”; the filter is the country name.

```57:92:client/src/components/DataPage.js
    const fetchCountries = async (query = '') => {
        try {
            const response = await axios.get('https://api.worldbank.org/v2/country?format=json');
            const countries = response.data[1].map(country => ({
                id: country.id,
                name: country.name,
                region: country.region.value,
                incomeLevel: country.incomeLevel.value,
                capitalCity: country.capitalCity,
                longitude: country.longitude,
                latitude: country.latitude,
            }));
    
            if (query) {
                
                const filteredCountries = countries.filter(country =>
                    country.name.toLowerCase().includes(query.toLowerCase())
                );
                setTopCountries(filteredCountries);
            } else {
                
                const shuffledCountries = countries.sort(() => Math.random() - 0.5);
                setTopCountries(shuffledCountries.slice(0, 5));
            }
        } catch (err) {
            console.error('Error fetching countries:', err);
        }
    };

    const [countrySearchQuery, setCountrySearchQuery] = useState('');

    const handleCountrySearchChange = (event) => {
        const query = event.target.value.trim();
        setCountrySearchQuery(query);
        fetchCountries(query);
    };
```

Topics. Every keystroke with a non-empty query refetches the full topic list and filters in the browser.

```149:174:client/src/components/DataPage.js
    const fetchTopics = async (query) => {
        try {
            const response = await axios.get('https://api.worldbank.org/v2/topic?format=json');
            const allTopics = response.data[1].map(topic => ({
                id: topic.id,
                name: topic.value,
                description: topic.sourceNote || 'No description available',
            }));
            const filteredTopics = allTopics.filter(topic =>
                topic.name.toLowerCase().includes(query.toLowerCase())
            );
            setTopics(filteredTopics);
        } catch (err) {
            console.error('Error fetching topics:', err);
        }
    };

    const handleTopicSearchChange = (event) => {
        const query = event.target.value.trim();
        setTopicSearchQuery(query);
        if (query.length > 0) {
            fetchTopics(query);
        } else {
            setTopics([]);
        }
    };
```

```http
GET https://api.worldbank.org/v2/country?format=json
GET https://api.worldbank.org/v2/topic?format=json
```

Envelope is `[metadata, rows]`. Both functions read index `1` and ignore metadata (page, total).

`Array.sort` here is in-place. `fetchTopCountries` and the empty-query branch of `fetchCountries` shuffle the mapped array with `Math.random() - 0.5`, which is not a uniform shuffle.

## 3. Flask handler and upstream HTTP

No Flask handler.

Upstream is the browser `GET` above. There is no indicator id, no date, and no per-country filter on the URL. The entire country list and the entire topic list come back on each search.

## 4. Response contract

- [../contracts/worldbank.country.mapped.json](../contracts/worldbank.country.mapped.json)
- [../contracts/worldbank.topic.mapped.json](../contracts/worldbank.topic.mapped.json)

## 5. JSX

```239:256:client/src/components/DataPage.js
                    <div className="topic-search">
                        <input
                            type="text"
                            placeholder="Search topics"
                            value={topicSearchQuery}
                            onChange={handleTopicSearchChange}
                            className="search-input"
                        />
                    </div>
                    <div className="company-search">
                            <input
                                type="text"
                                placeholder="Search regions"
                                value={countrySearchQuery}
                                onChange={handleCountrySearchChange}
                                className="search-input"
                            />
                        </div>
```

Topic results sit outside the `!searchQuery` block, so they stay visible while the user is searching companies:

```269:278:client/src/components/DataPage.js
            {topics.length > 0 && (
                <div className="topics-list">
                    {topics.map(topic => (
                        <div key={topic.id}>
                            <h4>{topic.name}</h4>
                            <p>{topic.description}</p>
                        </div>
                    ))}
                </div>
            )}
```

Country tiles are inside the signed-in, empty-company-search block:

```318:327:client/src/components/DataPage.js
                    <div className="top-countries">
                        {topCountries.map((country, index) => (
                            <div key={index} className="country-tile">
                                <h4>{country.name}</h4>
                                <p>Region: {country.region}</p>
                                <p>Income Level: {country.incomeLevel}</p>
                                <p>Capital City: {country.capitalCity}</p>
                            </div>
                        ))}
                    </div>
```

Typing in “Search companies” (`searchQuery`) hides the country tiles, the hint tiles, and the catalog results together.

## 6. Reproduce vs drop

Reproduce:

- Country tiles with name, region, income level, and capital.
- First paint of four countries from the full list if you want the same uncurated feel. A target product can show a stable set instead. That is a behavior change, not a hidden API.
- Topic name plus `sourceNote`.
- Client-side name filtering. The public API call does not take the search box value.

Drop:

- Longitude and latitude unless the new screen maps them. They are fetched and unused.
- The random shuffle, if the new screen should be deterministic.
- Any assumption that “Search regions” filters `region.value`. It filters `country.name`.
