# Trace: Career Assistant World Bank inputs

Topic, region, and report on `/career-assistant`. Topic and region are labels in the prompt. A selected report is catalog text in the prompt. None of the three fetch an indicator series.

News synthesis around this prompt is [04-career-assistant.md](04-career-assistant.md).

## 1. User action and route

Route: `/career-assistant`, signed in.

On mount the page loads topics, companies from Flask, regions, and reports.

- “Choose a Topic” is a `<select>` of topic **names**. The value stored is `inputs.selectedTopic`.
- “Choose a Region” is a `<select>` of `country.region.value` for every country. The same label appears many times. The value stored is `inputs.selectedRegion`.
- “Choose a Report” is a text input. Typing sets `inputs.selectedReport` and refetches `/api/search`. The list is filtered again in the browser. Clicking a row sets the input to the full title and reveals the HTML description plus “Read full report”.
- “Choose a Company” is the app’s `companies` table (`GET /companies`), not the World Bank. It becomes the prompt sentence `Consider the company: {name}.` It does not trigger NewsAPI by itself. NewsAPI still runs for favorites on the server.

The overview tells the user: “To query against data from The World Bank, use the Topic, Region, and Report sections.”

## 2. Client call

Mount:

```103:108:client/src/components/CareerAssistant.js
  useEffect(() => {
    fetchTopics();
    fetchCompanies();
    fetchRegions();
    fetchReports();
}, []);
```

`fetchReports()` is called with no argument, so `params.q` is `undefined`. Flask returns 400 when `q` is absent. Report search is meant to run from the input effect below.

Topics, names only:

```110:117:client/src/components/CareerAssistant.js
const fetchTopics = async () => {
    try {
        const response = await axios.get('https://api.worldbank.org/v2/topic?format=json');
        setTopics(response.data[1].map(topic => topic.value));
    } catch (err) {
        console.error('Error fetching topics:', err);
    }
};
```

Regions, not deduplicated:

```128:135:client/src/components/CareerAssistant.js
const fetchRegions = async () => {
    try {
        const response = await axios.get('https://api.worldbank.org/v2/country?format=json');
        setRegions(response.data[1].map(country => country.region.value));
    } catch (err) {
        console.error('Error fetching regions:', err);
    }
};
```

Report fetch and the effect that passes the typed string:

```137:149:client/src/components/CareerAssistant.js
useEffect(() => {
  if (inputs.selectedReport) {
      fetchReports(inputs.selectedReport);
  }
}, [inputs.selectedReport]);


const fetchReports = async (query) => {
  try {
      
      const response = await axios.get('http://localhost:5555/api/search', {
          params: { q: query }
      });
```

The rest of `fetchReports` matches the Data page: if `response.data.data` is a non-empty array, `setReports` to that array; otherwise `setReports([])`.

```http
GET https://api.worldbank.org/v2/topic?format=json
GET https://api.worldbank.org/v2/country?format=json
GET http://localhost:5555/api/search?q={selectedReport}
```

On submit, `selectedTopic`, `selectedRegion`, `selectedCompany`, and `selectedReport` are part of `inputs` and are posted with the Career Assistant body. See [04-career-assistant.md](04-career-assistant.md) for the POST.

## 3. Flask handler and upstream HTTP

Topic and region do not hit Flask until submit, and then only as strings:

```149:154:server/app.py
    if selected_topic:
        prompt += f"Focus on the topic: {selected_topic}. "
    if selected_company:
        prompt += f"Consider the company: {selected_company}. "
    if selected_region:
        prompt += f"Focus on the region: {selected_region}. "
```

Report branch. The server calls itself. The match is case-insensitive equality on the full title, not the client’s substring filter.

```155:179:server/app.py
    if selected_report:        
        response = requests.get(f"http://localhost:5555/api/search?q={selected_report}")
        if response.status_code == 200:
            data = response.json()
            reports = data.get('data', [])
            
            selected_report_normalized = selected_report.strip().lower()
            
            matching_report = next(
                (report for report in reports if report['title'].strip().lower() == selected_report_normalized),
                None
            )
            
            if matching_report:
                prompt += f"Search for related reports titled: {matching_report['title']}.\n"
                prompt += f"Description: {matching_report['description']}\n"
                
                if matching_report['link']:
                    prompt += f"Full Report Link: {matching_report['link']}\n"
                else:
                    prompt += "No link available for this report.\n"
            else:
                prompt += "No related report found.\n"
        else:
            prompt += "Failed to retrieve report data.\n"
```

`q` is interpolated into the URL with no encoding. A title with `&` or spaces depends on `requests` to keep the query intact only if the string is passed as `params`. Here it is part of the URL string.

Catalog proxy details: [06-data-page-catalog.md](06-data-page-catalog.md).

Indicator series are not requested on this path. `GET /api/world-bank` is [08-indicator-unused.md](08-indicator-unused.md).

## 4. Response contract

- Topics: [../contracts/worldbank.topic.mapped.json](../contracts/worldbank.topic.mapped.json)
- Regions: [../contracts/worldbank.country.mapped.json](../contracts/worldbank.country.mapped.json) (`careerAssistantRegions`)
- Catalog item and the self-call: [../contracts/datacatalog.search.json](../contracts/datacatalog.search.json)
- POST body fields: [../contracts/career-assistant.request.json](../contracts/career-assistant.request.json)

## 5. JSX

```463:494:client/src/components/CareerAssistant.js
        {/* Topic Dropdown */}
        <div className="search-field">
          <label>Choose a Topic</label>
          <select onChange={(e) => setInputs({ ...inputs, selectedTopic: e.target.value })} value={inputs.selectedTopic}>
            <option value="">Select Topic</option>
            {topics.map((topic, index) => (
              <option key={index} value={topic}>{topic}</option>
            ))}
          </select>
        </div>
  
        {/* Company Dropdown */}
        <div className="search-field">
          <label>Choose a Company</label>
          <select onChange={(e) => setInputs({ ...inputs, selectedCompany: e.target.value })} value={inputs.selectedCompany}>
            <option value="">Select Company</option>
            {companies.map((company, index) => (
              <option key={index} value={company.name}>{company.name}</option>
            ))}
          </select>
        </div>
  
        {/* Region Dropdown */}
        <div className="search-field">
          <label>Choose a Region</label>
          <select onChange={(e) => setInputs({ ...inputs, selectedRegion: e.target.value })} value={inputs.selectedRegion}>
            <option value="">Select Region</option>
            {regions.map((region, index) => (
              <option key={index} value={region}>{region}</option>
            ))}
          </select>
        </div>
```

Report field. The filter strips non-alphanumeric characters, lowercases, and checks `includes`. Clicking a row replaces the input with `filteredReport.title`, which then matches exactly and reveals the description.

```496:550:client/src/components/CareerAssistant.js
        {/* Report Search Field */}
        <div className="search-field">
          <label>Choose a Report</label>
          <input 
            type="text" 
            placeholder="Search reports..." 
            value={inputs.selectedReport} 
            onChange={(e) => {
              setInputs({ ...inputs, selectedReport: e.target.value });
            }} 
          />
          {/* Display filtered reports based on search */}
          {inputs.selectedReport && (
            <div className="report-search-results">
              {inputs.selectedReport && reports.filter(report => {
                const normalizedTitle = report.title.toLowerCase().replace(/[^a-z0-9]/g, '');
                const normalizedSearch = inputs.selectedReport.toLowerCase().replace(/[^a-z0-9]/g, '');
                return normalizedTitle.includes(normalizedSearch);
              }).length > 0 ? (
                reports.filter(report => {
                  const normalizedTitle = report.title.toLowerCase().replace(/[^a-z0-9]/g, '');
                  const normalizedSearch = inputs.selectedReport.toLowerCase().replace(/[^a-z0-9]/g, '');
                  return normalizedTitle.includes(normalizedSearch);
                }).map((filteredReport, index) => (
                  <div 
                    key={index} 
                    className="report-search-result-item"
                    onClick={() => setInputs({ ...inputs, selectedReport: filteredReport.title })}
                  >
                    <div>{filteredReport.title}</div>
                    {/* Initially Hide Description and Display After Selecting */}
                    {inputs.selectedReport === filteredReport.title && (
                      <div>
                        <div 
                          className="report-description" 
                          dangerouslySetInnerHTML={{ __html: filteredReport.description }}
                        ></div>
                        {/* Display Link if Available */}
                        {filteredReport.link && (
                          <div className="report-link">
                            <a href={filteredReport.link} target="_blank" rel="noopener noreferrer">
                              Read full report
                            </a>
                          </div>
                        )}
                      </div>
                    )}
                  </div>
                ))
              ) : (
                <div>No results found for "{inputs.selectedReport}".</div>
              )}
            </div>
          )}
        </div>
```

The user-facing answer is still the Career Assistant chat HTML from trace 04. Topic, region, and report do not get their own charts.

## 6. Reproduce vs drop

Reproduce if you need the same assistant behavior:

- A topic name and a region name as optional sentences in the model prompt.
- A catalog lookup whose title, description, and link are also sentences in that prompt.
- Client substring filtering that is looser than the server’s exact title match. The server only attaches a report when the submitted string equals a returned title after trim and lowercasing. Selecting a row does that. A partial typed string often becomes “No related report found.”

Replace:

- Duplicate region options. Deduping `region.value` changes the control, not the prompt sentence, as long as the selected string stays the same.
- The self-HTTP call to `http://localhost:5555/api/search`. Call the catalog function in-process. Hard-coding localhost and port 5555 fails when the API is hosted elsewhere.
- Unencoded `q` in that URL.
- HTML descriptions inserted into the prompt and into `dangerouslySetInnerHTML`.

Drop:

- The mount `fetchReports()` with no query. It 400s.
- Any expectation that these dropdowns plot `SP.POP.TOTL` or another indicator. They do not.
