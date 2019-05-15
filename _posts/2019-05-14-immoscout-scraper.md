---
layout: post
title: "Scrape immobilienscout24.de with in parallell with python"
---




I like webscraping. I find it amazing that with a bit of code one can collect whatever data from anywere in the web  and make use of it.
I am also interested in real estate, particularly in  the real estate market of my hometown Hamburg, Germany. 
To shed some light onto this market I decided to collect some data and play with it.

In this post I will run you through a basic webscraper in python.
I am using the libraries [requests](https://2.python-requests.org/en/master/) and [Beautifulsoup](https://www.crummy.com/software/BeautifulSoup/).
With [requests](https://2.python-requests.org/en/master/) I basically download the html document from the web that contains the information I am after.
With [Beautifulsoup](https://www.crummy.com/software/BeautifulSoup/) I extract that information.

In this post I will not waste too much time on details like how to come up with [Beautifulsoup](https://www.crummy.com/software/BeautifulSoup/) syntax to extract information.
I do not consider this a tutorial but a commented example of how to scrape [Immoscout](https://www.immobilienscout24.de/).

So lets get to it.



## The rough plan
In order to get all information on real estates for rent in Hamburg I first use the website  [Immoscout](https://www.immobilienscout24.de/) and enter our search criteria.
E.g. Flats for rent in Hamburg area.

That leads us to the first results page with matching classifieds. There you can see that certain data like classifieds ID and address are already shown.

So why not collect this data already. 
I will call this data "meta" data. 

On the bottom of the results page you see a dropdown menue with page numbers. One page on [Immoscout](https://www.immobilienscout24.de/) shows a maximum of 20
classifieds so I need to browse through all pages available to scrape all classifieds.

I use the pages information and the meta data to build a list of all available rent classifieds, to later scrape detail information for each of them.

When you now click on one of the flat offers, you will realize that the url looks like this `https://www.immobilienscout24.de/expose/90724026` plus a bunch of  parameters you can ignore for now.

That number in the end of the url is the classifieds ID (or expose ID).

Thats perfect, because now I can easily loop over our meta data that contains a lots of these ids, build URLs from them and scrape additional information.




## The Setup

First I import all the libraries that I need:

```python 
from bs4 import BeautifulSoup 
import time 
import requests 
import pandas as pd 
import re 
import datetime 
import numpy as np 
import multiprocessing 
from pebble 
import ProcessPool from 
concurrent.futures import TimeoutError



```

Then I define some functions that do the heavy lifting.

```python 

def get_soup(url, echo=False):
    
    if echo:
        print('scraping:', url)
    
    r = requests.get(url)
    data = r.text
    soup = BeautifulSoup(data, "lxml")
    return soup 



```	


Ok, so this first function uses requests to download the HTML document behind a given URL
and parses it so [Beautifulsoup](https://www.crummy.com/software/BeautifulSoup/) can apply its magic to it.


```python 
def parallelize_function_timeout(urls, func):
	with ProcessPool() as pool: 
		future = pool.map(func, urls, timeout=90) 
		iterator = future.result()
		result = pd.DataFrame()
		while True:
			try:
				result = result.append(next(iterator))
			except StopIteration:
				break
			except TimeoutError as error:
				print("scraping took longer than %d seconds" % error.args[1])
		return result 
		
		
		
```


The `parallelize_function_timeout` takes a list of inputs (URLs) and a function and runs this function in parallel using [multiprocessing](https://code.google.com/archive/p/python-multiprocessing/).

I use [multiprocessing](https://code.google.com/archive/p/python-multiprocessing/) here because scraping large amounts of URLs is a very I/O-Bound task. While it is not that computationally expesive the bottlenec really is I/O-Flow. To speed it up I will be scraping with multiple processes simultaneously.

So whats going on in this function?

First, I initiate a pool of subprocesses and run our function on the list of URLs with `future = pool.map(func, urls, timeout=90)` and specify a timeout duration in seconds. 
The results of all subprocesses running our function are appended into `results` and returned from the  `parallelize_function_timeout` function .If the iteration is stopped, e.g. by a keyboard interrupt it stops, if the function takes too long for one URL it times out and the URL is skipped.

We are now done with all the process management and can finally take care of the information retrieval.


```python 

def scrape_meta_chunk(url, return_soup=False):
    
    soup = get_soup(url)
    try:
        
        data = pd.DataFrame(
            [
            [location['data-result-id'],
             location.getText().split(', ')[-1],
             location.getText().split(', ')[-2],
             location.getText().split(', ')[-3] if len(location.getText().split(', ')) == 3 else '',
             str(datetime.datetime.now())]
            for location in
            soup.findAll("button", {"class": "button-link link-internal result-list-entry__map-link", "title": "Auf der Karte anzeigen"})], columns=['id','city_county', 'city_quarter', 'street', 'scraped_ts']
        )
        
    except Exception as err:
        print(url, err)
        data = pd.DataFrame()

    if return_soup:
        return data, soup
    else:
        return data 



```


This function extracts what I earlier called meta-data. 
It downloads a search results page from [Immoscout](https://www.immobilienscout24.de/) and extracts ID and addresses of all 20 listed results.

For that I use a python list comprehension to iterate through occurences of the HTML-Tag `<button>` with the class `button-link link-internal result-list-entry__map-link` and the title `Auf der Karte anzeigen`.
From each occurence (iteration) it retrieves the information we are after and stores it in a neat [pandas](https://pandas.pydata.org/) DataFrame.


```python 

def scrape_details_chunk(realty):
        
    expose_id = []
    title = []
    realty_type = []
    floor = []
    square_m = []
    storage_m = []
    num_rooms = []
    num_bedrooms = []
    num_baths = []
    num_carparks = []
    year_built = []
    last_refurb= []
    quality = []
    heating_type = []
    fuel_type = []
    energy_consumption = []
    energy_class = []
    net_rent = []
    gross_rent = []
    
            
    url = 'https://www.immobilienscout24.de/expose/' + realty
    
    soup= get_soup(url)
    expose_id.append(realty)
    title.append(soup.find('title').getText())
    if soup.find('dd', {'class':"is24qa-typ grid-item three-fifths"}):
        realty_type.append(soup.find('dd', {'class':"is24qa-typ grid-item three-fifths"}).getText().replace(' ', ''))
    else:
        realty_type.append(0)
    if soup.find('dd', {'class':"is24qa-etage grid-item three-fifths"}):
        floor.append(soup.find('dd', {'class':"is24qa-etage grid-item three-fifths"}).getText())
    else:
        floor.append(0)
    if soup.find('dd', {'class':"is24qa-wohnflaeche-ca grid-item three-fifths"}):
        square_m.append(soup.find('dd', {'class':"is24qa-wohnflaeche-ca grid-item three-fifths"}).getText().replace(' ', '').replace('m²', ''))
    else:
        square_m.append(0)
    if soup.find('dd', {'class':"is24qa-nutzflaeche-ca grid-item three-fifths"}):
        storage_m.append(soup.find('dd', {'class':"is24qa-nutzflaeche-ca grid-item three-fifths"}).getText().replace(' ', '').replace('m²', ''))
    else:
        storage_m.append(0)
    if soup.find('dd', {'class':"is24qa-zimmer grid-item three-fifths"}):
        num_rooms.append(soup.find('dd', {'class':"is24qa-zimmer grid-item three-fifths"}).getText().replace(' ', ''))
    else:
         num_rooms.append(0)
    if soup.find('dd', {'class':"is24qa-schlafzimmer grid-item three-fifths"}):
        num_bedrooms.append(soup.find('dd', {'class':"is24qa-schlafzimmer grid-item three-fifths"}).getText().replace(' ', ''))
    else:
        num_bedrooms.append(0)
    if soup.find('dd', {'class':"is24qa-badezimmer grid-item three-fifths"}):
        num_baths.append(soup.find('dd', {'class':"is24qa-badezimmer grid-item three-fifths"}).getText().replace(' ', ''))
    else:
        num_baths.append(0)
    if soup.find('dd', {'class':"is24qa-garage-stellplatz grid-item three-fifths"}):
        num_carparks.append(re.sub('[^0-9]', '', soup.find('dd', {'class':"is24qa-garage-stellplatz grid-item three-fifths"}).getText()))
    else:
        num_carparks.append(0)
    if soup.find('div', {'class':"is24qa-kaltmiete is24-value font-semibold"}):
        net_rent.append(re.sub('[^0-9]', '', soup.find('div', {'class':"is24qa-kaltmiete is24-value font-semibold"}).getText()))
    else:
        net_rent.append(0)
    if soup.find('dd', {'class':"is24qa-baujahr grid-item three-fifths"}):
        year_built.append(soup.find('dd', {'class':"is24qa-baujahr grid-item three-fifths"}).getText())
    else:
        year_built.append(0)
    if soup.find('dd', {'class':"is24qa-modernisierung-sanierung grid-item three-fifths"}):
        last_refurb.append(soup.find('dd', {'class':"is24qa-modernisierung-sanierung grid-item three-fifths"}).getText())
    else:
        last_refurb.append(0)
    if soup.find('dd', {'class':"is24qa-qualitaet-der-ausstattung grid-item three-fifths"}):
        quality.append(soup.find('dd', {'class':"is24qa-qualitaet-der-ausstattung grid-item three-fifths"}).getText().strip())
    else:
        quality.append(0)
    if soup.find('dd', {'class':"is24qa-heizungsart grid-item three-fifths"}):
        heating_type.append(soup.find('dd', {'class':"is24qa-heizungsart grid-item three-fifths"}).getText().strip())
    else:
        heating_type.append(0)
    if soup.find('dd', {'class':"is24qa-wesentliche-energietraeger grid-item three-fifths"}):
        fuel_type.append(soup.find('dd', {'class':"is24qa-wesentliche-energietraeger grid-item three-fifths"}).getText().strip())
    else:
        fuel_type.append(0)
    if soup.find('dd', {'class':"is24qa-endenergiebedarf grid-item three-fifths"}):
        energy_consumption.append(soup.find('dd', {'class':"is24qa-endenergiebedarf grid-item three-fifths"}).getText().strip())
    else:
        energy_consumption.append(0)
    if soup.find('dd', {'class':"is24qa-energieeffizienzklasse grid-item three-fifths"}):
        energy_class.append(soup.find('dd', {'class':"is24qa-energieeffizienzklasse grid-item three-fifths"}).getText().strip())
    else:
        energy_class.append(0)
    if soup.find('dd', {'class':"is24qa-gesamtmiete grid-item three-fifths font-bold"}):
        gross_rent.append(soup.find('dd', {'class':"is24qa-gesamtmiete grid-item three-fifths font-bold"}).getText().strip().replace(' €',''))
    else:
        gross_rent.append(0)
    results = pd.DataFrame({
    'id': expose_id,
    'title': title,
    'realty_type': realty_type,
    'floor': floor,
    'square_m': square_m,
    'storage_m': storage_m,
    'num_rooms': num_rooms,
    'num_bedrooms': num_bedrooms,
    'num_baths': num_baths,
    'num_carparks': num_carparks,
    'year_built': year_built,
    'last_refurb': last_refurb,
    'quality': quality,
    'heating_type': heating_type,
    'fuel_type': fuel_type,
    'energy_consumption': energy_consumption,
    'energy_class': energy_class,
    'net_rent': net_rent,
    'gross_rent': gross_rent,
    'scraped_ts': str(datetime.datetime.now())
    })
    
    return results



``` 


The function `scrape_details_chunk` is responsible for scraping the detailed information from each flats details page. This function comes to action when we loop throuh each classifieds ID using URLs like `https://www.immobilienscout24.de/expose/90724026`.

What it does in particular is this:
First it initiates empty lists for each piece or information we want to scrape like the classifieds title, square meters, number of rooms, you get the idea.
Then it creates the URL for a specific flat by concentrating the base URL ´https://www.immobilienscout24.de/expose/´ with the classifieds ID  from the meta-data.
It uses [Beautifulsoup](https://www.crummy.com/software/BeautifulSoup/) to parse and extract the data we are after and combines it into a DataFrame.


So much for setting everything up. Now we start scraping.


## The Scraping Process


Until now I explained how the scraping will be done and what functions I came up with to do the actual scraping.
Now we need to put all that to use.

First, we scrape the meta-data from the first result page of our real estate search. So we use the function `scrape_meta_chunk` on `https://www.immobilienscout24.de/Suche/S-T/P-1/Wohnung-Miete/Umkreissuche/Hamburg/-/1840/2621814/-/-/30`.


```python


url = 'https://www.immobilienscout24.de/Suche/S-T/P-1/Wohnung-Miete/Umkreissuche/Hamburg/-/1840/2621814/-/-/30' 

realty_meta_df, soup = scrape_meta_chunk(url, return_soup=True) 

num_pages = len(soup.find_all('option')) 

print('scraped', len(realty_meta_df), 'realties on first page') 
print('found', num_pages, 'pages to scrape') 


```
    > scraped 20 realties on first page
    > found 101 pages to scrape 

With `num_pages = len(soup.find_all('option'))` we extract the number of result pages for this search so we can loop over each page later.
Now that we know how many pages there are with our search results (101 in this case) we can start looping over them and extract the meta-data from each of them.


```python


realty_meta_urls = ['https://www.immobilienscout24.de/Suche/S-T/P-' + str(page) + '/Wohnung-Miete/Umkreissuche/Hamburg/-/1840/2621814/-/-/30' for page in range(2, num_pages + 1)] realty_meta_df = 
realty_meta_df.append(parallelize_function_timeout(realty_meta_urls, scrape_meta_chunk))
print('Done')
 
    
      
```
    > Done 

Here we crate a new URL for each page number we have found. This gives us a list of 101 pages to scrape meta-data from.
We pass this list and the function `realty_meta_urls` to the `parallelize_function_timeout` to scrape these URLs on multiple cores.
This gives us a DataFrame containing meta-data for all found classifieds including their IDs.
With that we are able to scrape each detail page as follows.



```python


realty_details = pd.DataFrame(
columns=[
    'expose_id',
    'title',
    'realty_type',
    'floor',
    'square_m',
    'storage_m',
    'num_rooms',
    'num_bedrooms',
    'num_baths',
    'num_carparks',
    'year_built',
    'last_refurb',
    'quality',
    'heating_type',
    'fuel_type',
    'energy_consumption',
    'energy_class',
    'net_rent',
    'gross_rent' ])
	
	

realties = list(set(list(realty_meta_df.id))) 

print('scraping', len(realties), 'realties...') 
realty_details_df = parallelize_function_timeout(realties, scrape_details_chunk)
realty_details_df = realty_details_df.drop_duplicates() 
realty_details_df = realty_details_df.reset_index(drop=True) 

print('scraped', len(realty_details_df), 'realties.') 



```
    > scraping 2020 realties...
    > scraped 2020 realties. 


Here I am creating a new DataFrame with a column for each data dimension I am scraping. Then I pass a list of classifieds IDs to the `scrape_details_chunk` via `parallelize_function_timeout`. This takes a while and finally returns a clean DataFrame containing all desired detail-data.
At last I merge the detail-data with the meta-data to end up with only one DataFrame and print the first few rows of the scraped data.

Thats it. We are done.

	
```python 

realty_meta_df['id'] = realty_meta_df['id'].astype(int) 
realty_details_df['id'] = realty_details_df['id'].astype(int) 

final_data = realty_meta_df.drop('scraped_ts',axis = 1).merge(realty_details_df, on='id', suffixes = ['_meta', '_details']) 


 
final_data.head() 
``` 



<table>
  <thead>
    <tr>
      <th></th>
      <th>id</th>
      <th>city_county</th>
      <th>city_quarter</th>
      <th>street</th>
      <th>title</th>
      <th>realty_type</th>
      <th>floor</th>
      <th>square_m</th>
      <th>storage_m</th>
      <th>num_rooms</th>
      <th>...</th>
      <th>year_built</th>
      <th>last_refurb</th>
      <th>quality</th>
      <th>heating_type</th>
      <th>fuel_type</th>
      <th>energy_consumption</th>
      <th>energy_class</th>
      <th>net_rent</th>
      <th>gross_rent</th>
      <th>scraped_ts</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>110713974</td>
      <td>Hamburg</td>
      <td>Altona-Nord</td>
      <td>Glückel-von-Hameln-Straße 2</td>
      <td>neue Wohnung - neues Glück</td>
      <td>Etagenwohnung</td>
      <td>0</td>
      <td>75,14</td>
      <td>0</td>
      <td>2</td>
      <td>...</td>
      <td>2019</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1150</td>
      <td>1.370,66</td>
      <td>2019-05-14 13:30:01.252678</td>
    </tr>
    <tr>
      <th>1</th>
      <td>110714010</td>
      <td>Hamburg</td>
      <td>Altona-Nord</td>
      <td>Susanne-von-Paczensky-Straße 9</td>
      <td>Grandiose 4-Zimmerwohung! ***Erstbezug***</td>
      <td>Etagenwohnung</td>
      <td>0</td>
      <td>99,39</td>
      <td>0</td>
      <td>4</td>
      <td>...</td>
      <td>2019</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1440</td>
      <td>1.828,13</td>
      <td>2019-05-14 13:29:57.649693</td>
    </tr>
    <tr>
      <th>2</th>
      <td>110247048</td>
      <td>Hamburg</td>
      <td>Altona-Nord</td>
      <td>Susanne-von-Paczensky-Straße 7</td>
      <td>Traumhafte Penthousewohnung! ***Erstbezug***</td>
      <td>Etagenwohnung</td>
      <td>0</td>
      <td>166,02</td>
      <td>0</td>
      <td>5</td>
      <td>...</td>
      <td>2019</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>2500</td>
      <td>3.090,15</td>
      <td>2019-05-14 13:29:32.219968</td>
    </tr>
    <tr>
      <th>3</th>
      <td>110247042</td>
      <td>Hamburg</td>
      <td>Altona-Nord</td>
      <td>Susanne-von-Paczensky-Straße 11</td>
      <td>Geniale 3-Zimmerwohnung! ***Erstbezug***</td>
      <td>Etagenwohnung</td>
      <td>0</td>
      <td>97,65</td>
      <td>0</td>
      <td>3</td>
      <td>...</td>
      <td>2019</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1470</td>
      <td>1.854,57</td>
      <td>2019-05-14 13:29:44.507045</td>
    </tr>
    <tr>
      <th>4</th>
      <td>110714047</td>
      <td>Hamburg</td>
      <td>Altona-Nord</td>
      <td>Eva-Rühmkorf-Straße 8</td>
      <td>***Hervorragende 3-Zimmerwohnung*** Erstbezug!</td>
      <td>Etagenwohnung</td>
      <td>0</td>
      <td>86,04</td>
      <td>0</td>
      <td>3</td>
      <td>...</td>
      <td>2019</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1300</td>
      <td>1.648,85</td>
      <td>2019-05-14 13:29:47.011108</td>
    </tr>
  </tbody> </table> 