# LoadCurrencies
Here I will describe how to automatically load currency exchange rates in case you don't have a specific API service for it. 

## European Central Bank
Let's have a look on data published on official ECB website on EUR/USD exchange rate. [ECB USD Statistics](https://www.ecb.europa.eu/stats/policy_and_exchange_rates/euro_reference_exchange_rates/html/eurofxref-graph-usd.en.html)
<img src="/img/ECB_screenshot_1.JPG" alt="ECBscreenshot" width="50%"/>
This graph ha all the information we need. Let's sort out how to get it in a format we can use for analysis. 

## Getting the data
Process involves a tiny bit of research. We need to see the `html` code of the page. 
Scrolling deeper into the code one can find a function:
```javascript
generateChartData()
{
 var chartData = new Array();
chartData.push({ date: new Date(1999,0,4), rate: 1.1789 });
chartData.push({ date: new Date(1999,0,5), rate: 1.179 });
chartData.push({ date: new Date(1999,0,6), rate: 1.1743 });
.................
chartDataInverse.push({ date: new Date(2024,10,14), rate: 0.9494 });
chartDataInverse.push({ date: new Date(2024,10,15), rate: 0.9449 });
chartDataInverse.push({ date: new Date(2024,10,18), rate: 0.9477 });

return chartDataInverse;
}
```
Voila! It has all the necessary data for us, but how can we parse it. Well, for that I prefer to use `Python` script. Please note, that due to the nature of request you may get an `InsecureRequestWarning`. We can suppress it using `urllib3` library:

```Python
import requests
import pandas as pd
from datetime import datetime as dt, timedelta as td
from bs4 import BeautifulSoup
import re
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_eur_usd(url):
    response = requests.get(url, verify=False)

    if response.status_code==200:
        #print('Status code 200')
        soup = BeautifulSoup(response.content, 'html.parser')
        script = soup.select('script')[3]

        pattern = r"chartData\.push\(\{\s*date:\s*new Date\((\d+),(\d+),(\d+)\),\s*rate:\s*([\d.]+)\s*\}\);"
        matches = re.findall(pattern, script.string)
        chart_data = []
        for match in matches:
            year, month, day, rate = match
            date = f"{year}-{int(month)+1:02d}-{int(day):02d}"  # Convert JS month (0-based) to 1-based
            chart_data.append({"date": date, "rate": float(rate)})

        df = pd.DataFrame(chart_data)
        df.date = pd.to_datetime(df.date).dt.date
        #print("Data loaded!")
        return df
    else:
        print(f"Status code: {response.status_code}, Text: {response.text}")
```
