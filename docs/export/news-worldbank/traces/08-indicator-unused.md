# Trace: World Bank indicator proxy (unused)

Server route with no UI. Included so a migration does not invent a screen for it, and so a later port can wire it on purpose.

## 1. User action and route

No React route calls this endpoint.

Data page copy says “Access data and reporting from The World Bank.” That tile sits next to catalog results. It does not call `/api/world-bank`.

A repository search for `world-bank` and `SP.POP.TOTL` hits `server/app.py` only.

## 2. Client call

None.

## 3. Flask handler and upstream HTTP

```206:220:server/app.py
@app.route("/api/world-bank", methods=["GET"])
def get_world_bank_data():
    indicator = request.args.get("indicator", "SP.POP.TOTL")
    url = f"http://api.worldbank.org/v2/country/all/indicator/{indicator}?format=json"
    response = requests.get(url)

    if response.status_code != 200:
        return jsonify({"error": "Failed to fetch World Bank data"}), 500

    data = response.json()

    if len(data) < 2 or not data[1]:
        return jsonify({"error": f"No data found for indicator {indicator}"}), 500

    return jsonify(data)
```

```http
GET /api/world-bank
GET /api/world-bank?indicator=SP.POP.TOTL
GET http://api.worldbank.org/v2/country/all/indicator/{indicator}?format=json
```

Default indicator `SP.POP.TOTL` is total population. The upstream URL uses `http`, not `https`. `country/all` is every country. There is no `per_page`, `date`, or `mrv` argument, so the call uses World Bank defaults for page size and date range.

Success returns the raw envelope: a JSON array whose second element is the observation list. The handler does not map `country`, `date`, or `value`.

## 4. Response contract

[../contracts/worldbank.indicator.json](../contracts/worldbank.indicator.json)

## 5. JSX

None.

## 6. Reproduce vs drop

Drop this route if the new product only needs what users see today: country tiles, topic blurbs, catalog reports, and prompt labels.

Reproduce it only when the new screen needs a time series. In that case the current handler is a thin proxy:

- Query param `indicator`, default `SP.POP.TOTL`.
- Pass through the World Bank array.
- 500 when the upstream fails or `data[1]` is empty.

A real chart would add `date`, paging, and a field map (`country.value`, `date`, `value`) that this handler does not implement. That would be new behavior, not a port of a screen.
