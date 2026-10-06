# Trace: Career Assistant news and synthesis

Two layers. The client tries to send news. The server ignores that payload and builds its own prompt from favorites, NewsAPI, and optional World Bank text.

World Bank dropdowns and the catalog self-call are in [07-career-assistant-world-bank.md](07-career-assistant-world-bank.md). This file is the news path and the prompt shell those fields drop into.

## 1. User action and route

Route: `/career-assistant`. The nav link renders only when `isLoggedIn` is set.

Signed out, the page returns “You must be logged in to use the Career Assistant.” before the form.

Signed in, the user sets a prompt and optional controls, then clicks “Ask Career Assistant”. The answer appears above the form. `formatResponse` turns bare URLs into anchors. A copy button writes HTML to the clipboard.

On-screen instruction (the product’s own description of news and catalog links):

```339:341:client/src/components/CareerAssistant.js
  <p className="highlight italic-text">
    Career Assistant will include news report links for favorite companies and report links from The World Bank Data Catalog when available. Please specifically request these resources in your prompt to ensure that they are included.
  </p>
```

Preferred News Sources are checkboxes: Reuters, Bloomberg, The Verge, TechCrunch, Local News Sources. Time Frame radios: Last 7 days, Last 30 days, Last 6 months. Default time frame is “Last 30 days”.

## 2. Client call

Favorites prefetch expects a wrapper the API does not return:

```38:50:client/src/components/CareerAssistant.js
  const fetchUserFavorites = async (userId) => {
    try {
      const response = await axios.get(`http://localhost:5555/favorites?user_id=${userId}`);
      if (response.data && response.data.favorites) {
        setUserFavorites(response.data.favorites);
      } else {
        setUserFavorites([]);
      }
    } catch (error) {
      setError('Failed to load favorites');
      console.error('Error fetching user favorites:', error);
    }
  };
```

News prefetch uses fields that array does not have. It does not run while `userFavorites` stays empty.

```52:68:client/src/components/CareerAssistant.js
  const fetchNewsArticles = async () => {
    const newsPromises = userFavorites.map(async (favorite) => {
      try {
        
        const response = await axios.get(`http://localhost:5555/news/${favorite.id}`);
        if (response.data && response.data.articles) {
          return {
            company_name: favorite.name,
            articles: response.data.articles,
          };
        }
        return { company_name: favorite.name, articles: [] };
      } catch (error) {
        console.error('Error fetching news for', favorite.name, error);
        return { company_name: favorite.name, articles: [] };
      }
    });
```

`useEffect` on `userFavorites` and `inputs.time_frame` calls `fetchNewsArticles` only when `userFavorites.length > 0`. Changing the time frame does not change the NewsAPI query. It only re-runs this effect.

Submit still posts. `companyNames` is built from the empty client list, so the extra sentence lists no companies. `user_id` is what makes the server path work.

```177:190:client/src/components/CareerAssistant.js
  const handleSubmit = async () => {
    try {
      
      const companyNames = userFavorites.map(favorite => favorite.name).join(', ');
  
      const payload = {
        ...inputs,
        user_favorites: userFavorites,
        news_articles: newsArticles,
        user_id: loggedInUser,
        prompt: `${inputs.prompt} Please check if any of the following companies are mentioned: ${companyNames}. If any of these companies are mentioned, provide the latest news articles and ensure that the links to these articles are clickable.`,
      };
  
      const res = await axios.post('http://localhost:5555/career-assistant', payload);
```

```http
GET  http://localhost:5555/favorites?user_id={userId}
GET  http://localhost:5555/news/{favorite.id}    (not reached with current favorites JSON)
POST http://localhost:5555/career-assistant
```

## 3. Flask handler and upstream HTTP

Auth and server-side news. The handler never reads `news_articles` or `user_favorites` from the body.

```63:101:server/app.py
@app.route('/career-assistant', methods=['POST'])
def career_assistant():
    data = request.get_json()
    logging.debug(f"Received data: {data}") 
    
    user_id = data.get('user_id')
    
    if not user_id:
        logging.warning("User not logged in.")
        return jsonify({"message": "User not logged in."}), 401
    
    
    favorites = Favorites.query.filter_by(user_id=user_id).all()
    logging.debug(f"Fetched favorites for user {user_id}: {favorites}")
    
    favorite_companies = [{"company_name": fav.company.name, 
                           "link": fav.company.link, 
                           "category": fav.company.category.name if fav.company.category else None,
                           "id": fav.company.id} for fav in favorites]
    
    
    news_details = []
    for company in favorite_companies:
        articles = fetch_news_for_company(company['company_name'], desired_article_count=3)
        logging.debug(f"Fetched articles for company {company['company_name']}: {articles}")
        news_details.append({
            "company_name": company['company_name'],
            "news_articles": [{"title": article['title'], "url": article['url']} for article in articles]
        })

    
    favorites_prompt = ""
    if favorite_companies:
        favorites_prompt = "User's favorite companies:\n" + "\n".join(
            [f"- {company['company_name']} (Category: {company['category']})" for company in favorite_companies]
        ) + "\n\n"
        favorites_prompt += "Recent news articles:\n" + "\n".join(
            [f"{news['company_name']}:\n" + "\n".join([f"  - <a href='{article['url']}' target='_blank'>{article['title']}</a>" for article in news['news_articles']]) for news in news_details]
        ) + "\n"
```

Prompt fields that look like news filters and are only text:

```119:146:server/app.py
    scope_of_analysis = data.get('scope_of_analysis', [])
    sentiment_tone = data.get('sentiment_tone', 'Neutral')
    level_of_detail = data.get('level_of_detail', 'Brief')
    preferred_sources = data.get('preferred_sources', [])
    time_frame = data.get('time_frame', 'Last 30 days')
    industry_focus = data.get('industry_focus', [])
    specific_topics = data.get('specific_topics', '')
    preferred_format = data.get('preferred_format', 'Bullet Points')

    selected_topic = data.get('selectedTopic', '')
    selected_company = data.get('selectedCompany', '')
    selected_region = data.get('selectedRegion', '')
    selected_report = data.get('selectedReport', '')
    
    
    prompt += f"Scope of Analysis: {', '.join(scope_of_analysis)}\n"
    prompt += f"Sentiment Tone: {sentiment_tone}\n"
    prompt += f"Level of Detail: {level_of_detail}\n"
    if preferred_sources:
        prompt += f"Preferred News Sources: {', '.join(preferred_sources)}\n"
    prompt += f"Time Frame: {time_frame}\n"
    if industry_focus:
        prompt += f"Industry Focus: {', '.join(industry_focus)}\n"
    if specific_topics:
        prompt += f"Specific Topics: {specific_topics}\n"
    prompt += f"Preferred Format: {preferred_format}\n"
    prompt += f"Recent company news articles: Include clickable links to the latest news articles **only** for the companies mentioned in the user's career inquiry. If the company is part of the user's favorites but is not mentioned in the inquiry, exclude it from the news section.\n"
    prompt += favorites_prompt
```

OpenAI call and response key:

```183:201:server/app.py
    api_data = {
        "model": "gpt-4o-mini",
        "messages": [{"role": "user", "content": prompt}],
        "temperature": 0.7
    }

    try:
        response = requests.post("https://api.openai.com/v1/chat/completions", 
                                 headers={"Content-Type": "application/json", 
                                          "Authorization": f"Bearer {OPENAI_API_KEY}"}, 
                                 json=api_data)
        response.raise_for_status()
        
        
        api_response = response.json()
        logging.debug(f"API response: {api_response}")
        
        ai_response = api_response['choices'][0]['message']['content'].strip()
        return jsonify({"response": ai_response}), 200
```

News upstream is [01-newsapi-choke-point.md](01-newsapi-choke-point.md) with count 3. Headlines for every favorite are in the prompt. The model is told to emit links only for companies the user mentioned.

## 4. Response contract

[../contracts/career-assistant.request.json](../contracts/career-assistant.request.json)

Success body is `{ "response": "<text>" }`. The client reads `res.data.response`, not `answer`.

## 5. JSX

Sources and time frame. These values ride along in `inputs` and become prompt lines. They do not change `fetch_news_for_company`.

```409:443:client/src/components/CareerAssistant.js
        {/* Preferred News Sources */}
        <div className="prompt-criteria">
          <label>Preferred News Sources</label>
          <div>
            {['Reuters', 'Bloomberg', 'The Verge', 'TechCrunch', 'Local News Sources'].map((option) => (
              <label key={option}>
                <input
                  type="checkbox"
                  value={option}
                  checked={inputs.preferred_sources.includes(option)}
                  onChange={(e) => handleInputChange(e, 'preferred_sources')}
                />
                {option}
              </label>
            ))}
          </div>
        </div>
  
        {/* Time Frame */}
        <div className="prompt-criteria">
          <label>Time Frame</label>
          <div>
            {['Last 7 days', 'Last 30 days', 'Last 6 months'].map((option) => (
              <label key={option}>
                <input
                  type="radio"
                  value={option}
                  checked={inputs.time_frame === option}
                  onChange={(e) => handleInputChange(e, 'time_frame')}
                />
                {option}
              </label>
            ))}
          </div>
        </div>
```

Answer rendering:

```226:241:client/src/components/CareerAssistant.js
  const formatResponse = (response) => {
    
    response = response.replace(/(https?:\/\/[^\s]+)(?=\s|$|[.,;?!)(]*)(?=\b)/g, (match) => {
        return `<a href="${match}" target="_blank">${match}</a>`;
    });

    
    response = response.replace(/\* (.*?)\n/g, '<ul><li>$1</li></ul>');
    response = response.replace(/^(#) (.*?)$/gm, '<h1>$2</h1>');
    response = response.replace(/^(##) (.*?)$/gm, '<h2>$2</h2>');
    response = response.replace(/^(###) (.*?)$/gm, '<h3>$2</h3>');
    response = response.replace(/^(####) (.*?)$/gm, '<h4>$2</h4>');
    response = response.replace(/\n/g, '<br />');

    return response;
};
```

```287:300:client/src/components/CareerAssistant.js
      <div className="output-section">
        <h3>Career Assistant's Response</h3>
        <div className="chat-window">
          {responses.map((response, index) => (
            <div key={index}>
              <div>
                <strong>Q:</strong> {response.question}
              </div>
              <div dangerouslySetInnerHTML={{ __html: formatResponse(response.answer) }} />
              {/* Copy Button */}
              <button onClick={() => copyToClipboard(response.answer)}>Copy Response</button>
            </div>
          ))}
        </div>
```

`response.question` is `inputs.prompt` from before the submit overwrote `payload.prompt`. The stored question is the textarea text, not the sentence that appends company names.

## 6. Reproduce vs drop

Reproduce if the new product should match current behavior:

- Require `user_id` and load favorites from your own store.
- Fetch three headlines per favorite company name and place title links in the model context.
- Instruct the model to attach those links only when the question mentions the company.
- Keep Preferred News Sources and Time Frame as prompt instructions unless you deliberately wire them to NewsAPI `sources` and `from`/`to`.
- Render the model text as HTML links.

Drop, or fix on purpose:

- The client favorites check for `response.data.favorites` and the `favorite.id` / `favorite.name` prefetch. They do not affect the answer today.
- Sending `news_articles` in the POST body. The server does not read it.
- The localhost self-call for catalog reports. That belongs to the World Bank trace, and it assumes Flask is on port 5555 on the same machine.

`formatResponse` runs a URL regex and then replaces every newline with `<br />`. Anchors the server already embedded as `<a href='...'>` are not rewritten by the URL regex. A target renderer can keep that behavior or replace it with a markdown renderer. Either way, treat the model output as HTML.
