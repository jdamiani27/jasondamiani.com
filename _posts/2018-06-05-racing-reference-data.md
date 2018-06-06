---
layout: post
title:  "Scraping Racing Reference for NASCAR Data"
date:   2018-06-05 22:43:00 -0400
---


[Racing Reference](http://racing-reference.info) is the comprehensive source for up-to-date historical NASCAR race results.  In addition, statistics are recorded by driver, owner, and crew chief, making for an awesome foundation of racing data analysis.

Traditional stick and ball sports have seen their data and analytics movement, with a variety of tools to get data from authoritative sources into the hands of eager sports data geeks.  NASCAR and racing in general have seemingly been behind on this movement, lacking the open source foundations to unleash the potential of years of historical data in tools like Python and the PyData stack.

I am planning to eventually develop a full fledged Python package to make it easier to retrieve structured NASCAR data with a few parameters.  This post covers an attempt to begin to bridge that gap by scraping modern era race results in the Monster Energy NASCAR Cup Series from Racing Reference.  

## Making Requests
To get started, we'll import the following four Python packages:
* requests - makes our HTTP requests
* BeautifulSoup - aids in parsing and searching HTML
* re - the Python standard regex module; useful for finding strings and extracting data from those strings
* pandas - the swiss army knife of working with data in Python; provides us with tools to get data into a nice, tidy format for analysis


```python
import requests
from bs4 import BeautifulSoup
import re
import pandas as pd


BASE_URL = 'http://racing-reference.info'
```

Racing Reference uses a RESTful url scheme for accessing various pages.  The raceyear endpoint contains high level results of all the races in a given year.  For instance, we can receive a page with results for all races in 1979 at this url:

    http://racing-reference.info/raceyear/1979/W

'W', I believe, stands for Winston Cup, the long-time name of the top level of NASCAR competition.  In addition to the date, site, winner, and other race details, the raceyear page contains links to the full details of each race.  We'll need these links later.

First, we'll request each raceyear from 1979 to present (2018); the NASCAR "modern era"


```python
years = range(1979, 2019)

cup_results = [requests.get(BASE_URL + f'/raceyear/{year}/W') for year in years]
```

Checking HTTP status codes of our responses to make sure all our requests were successful


```python
set([r.status_code for r in cup_results])
```




    {200}



Each of these yearly cup series results pages has links to race details.  Race detail pages have url's like this:

    http://racing-reference.info/race/1979_Winston_Western_500/W

The yearly results pages have HTML `<a>` tags with relative links.  Lets find them all


```python
race_anchors = []
href_regex = re.compile('/race/.*/W')

for c in cup_results:
    race_anchors.extend(BeautifulSoup(c.text, 'lxml').find_all(href=href_regex))
```


```python
race_anchors[:5]
```




    [<a href="/race/1979_Winston_Western_500/W" title="Winston Western 500">1</a>,
     <a href="/race/1979_Daytona_500/W" title="Daytona 500">2</a>,
     <a href="/race/1979_Carolina_500/W" title="Carolina 500">3</a>,
     <a href="/race/1979_Richmond_400/W" title="Richmond 400">4</a>,
     <a href="/race/1979_Atlanta_500/W" title="Atlanta 500">5</a>]



We can now use the href attribute of these `<a>` tags to build a full url to request the race detail pages


```python
races = [requests.get(BASE_URL + a.attrs['href']) for a in race_anchors]
```

Again, checking the status codes.  All 200s is what we're after


```python
set([r.status_code for r in races])
```




    {200}



## Extracting Race Results

To extract the race results stored as an HTML table, we can use the Pandas read_html function.

Given the text of the page, read_html will return a list of dataframes from all tables found.  We can filter by using the match argument to find tables containing the provided string or regex


```python
[df.shape for df in pd.read_html(races[0].text, match='Sponsor / Owner', header=0)]
```




    [(83, 398), (35, 11)]



A list of two dataframes was returned.  This is due to the nesting of tables in the structure of the race pages.

The last element of the returned list is what we're after


```python
pd.read_html(races[0].text, match='Sponsor / Owner', header=0)[-1].head()
```




<div class="dataframe-container">
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Fin</th>
      <th>St</th>
      <th>#</th>
      <th>Driver</th>
      <th>Sponsor / Owner</th>
      <th>Car</th>
      <th>Laps</th>
      <th>Money</th>
      <th>Status</th>
      <th>Led</th>
      <th>Pts</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>4</td>
      <td>88</td>
      <td>Darrell Waltrip</td>
      <td>Gatorade (DiGard Racing)</td>
      <td>Chevrolet</td>
      <td>119</td>
      <td>21150</td>
      <td>running</td>
      <td>87</td>
      <td>185</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>1</td>
      <td>21</td>
      <td>David Pearson</td>
      <td>Purolator (Wood Brothers)</td>
      <td>Mercury</td>
      <td>119</td>
      <td>14200</td>
      <td>running</td>
      <td>9</td>
      <td>175</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>2</td>
      <td>11</td>
      <td>Cale Yarborough</td>
      <td>Busch (Junior Johnson)</td>
      <td>Oldsmobile</td>
      <td>119</td>
      <td>12675</td>
      <td>running</td>
      <td>3</td>
      <td>170</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>8</td>
      <td>73</td>
      <td>Bill Schmitt</td>
      <td>Old Milwaukee (Bill Schmitt)</td>
      <td>Oldsmobile</td>
      <td>118</td>
      <td>8000</td>
      <td>running</td>
      <td>0</td>
      <td>160</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>17</td>
      <td>1</td>
      <td>Donnie Allison</td>
      <td>Hawaiian Tropic (Hoss Ellington)</td>
      <td>Chevrolet</td>
      <td>118</td>
      <td>7550</td>
      <td>running</td>
      <td>0</td>
      <td>155</td>
    </tr>
  </tbody>
</table>
</div>



## Extracting Race Details
To help with analysis, it will be useful to extract some further details about the race; laps, track length, track type, and race length in particular


```python
r_details = re.compile(r'(\d+) laps\*? on a (\d?\.\d{3}) mile (.*) \((\d+\.\d+) miles\)')
details_match = r_details.search(races[0].text)
details_match[0]
```




    '119 laps on a 2.620 mile road course (311.8 miles)'



In addition to matching the entire string, our regex captured the following


```python
details_match[1], details_match[2], details_match[3], details_match[4]
```




    ('119', '2.620', 'road course', '311.8')



Furthermore, we can simply use the url to extract the year and race


```python
races[0].url
```




    'http://racing-reference.info/race/1979_Winston_Western_500/W'




```python
race_id = races[0].url.split('/')[-2]
race_id
```




    '1979_Winston_Western_500'




```python
r_race_id = re.compile(r'(\d{4})_(.*)')
race_id_match = r_race_id.search(race_id)
race_id_match[1], race_id_match[2]
```




    ('1979', 'Winston_Western_500')



It would also be useful to extract the name of the track.  We'll again use a regex to find the url pattern of Racing Reference track pages like:

    http://racing-reference.info/tracks/Riverside_International_Raceway


```python
r_track_name = re.compile('/tracks/.*')
BeautifulSoup(races[0].text, 'lxml').find(href=r_track_name).text
```




    'Riverside International Raceway'



Putting it all together to create a dataframe for each modern era race


```python
race_data_frames = []

for r in races:
    df = pd.read_html(r.text, match='Sponsor / Owner', header=0)[-1]

    details_match = r_details.search(r.text)
    df['race_length_laps'] = int(details_match[1])
    df['track_length_miles'] = float(details_match[2])
    df['track_type'] = details_match[3]
    df['race_length_miles'] = float(details_match[4])

    race_id = r.url.split('/')[-2]
    race_id_match = r_race_id.search(race_id)
    df['year'] = int(race_id_match[1])
    df['race_name'] = race_id_match[2]

    df['track_name'] = BeautifulSoup(r.text, 'lxml').find(href=r_track_name).text

    race_data_frames.append(df)
```


```python
race_data_frames[-1].head()
```




<div class="dataframe-container">
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Fin</th>
      <th>St</th>
      <th>#</th>
      <th>Driver</th>
      <th>Sponsor / Owner</th>
      <th>Car</th>
      <th>Laps</th>
      <th>Status</th>
      <th>Led</th>
      <th>Pts</th>
      <th>PPts</th>
      <th>race_length_laps</th>
      <th>track_length_miles</th>
      <th>track_type</th>
      <th>race_length_miles</th>
      <th>year</th>
      <th>race_name</th>
      <th>track_name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1</td>
      <td>4</td>
      <td>78</td>
      <td>Martin Truex, Jr.</td>
      <td>Bass Pro Shops / 5-hour Energy (Barney Visser)</td>
      <td>Toyota</td>
      <td>160</td>
      <td>running</td>
      <td>31</td>
      <td>57</td>
      <td>6</td>
      <td>160</td>
      <td>2.5</td>
      <td>paved track</td>
      <td>400.0</td>
      <td>2018</td>
      <td>Pocono_400</td>
      <td>Pocono Raceway</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2</td>
      <td>13</td>
      <td>42</td>
      <td>Kyle Larson</td>
      <td>DC Solar (Chip Ganassi)</td>
      <td>Chevrolet</td>
      <td>160</td>
      <td>running</td>
      <td>0</td>
      <td>43</td>
      <td>0</td>
      <td>160</td>
      <td>2.5</td>
      <td>paved track</td>
      <td>400.0</td>
      <td>2018</td>
      <td>Pocono_400</td>
      <td>Pocono Raceway</td>
    </tr>
    <tr>
      <th>2</th>
      <td>3</td>
      <td>5</td>
      <td>18</td>
      <td>Kyle Busch</td>
      <td>M&amp;M's Red White &amp; Blue (Joe Gibbs)</td>
      <td>Toyota</td>
      <td>160</td>
      <td>running</td>
      <td>13</td>
      <td>51</td>
      <td>0</td>
      <td>160</td>
      <td>2.5</td>
      <td>paved track</td>
      <td>400.0</td>
      <td>2018</td>
      <td>Pocono_400</td>
      <td>Pocono Raceway</td>
    </tr>
    <tr>
      <th>3</th>
      <td>4</td>
      <td>2</td>
      <td>4</td>
      <td>Kevin Harvick</td>
      <td>Busch Beer (Stewart Haas Racing)</td>
      <td>Ford</td>
      <td>160</td>
      <td>running</td>
      <td>89</td>
      <td>52</td>
      <td>1</td>
      <td>160</td>
      <td>2.5</td>
      <td>paved track</td>
      <td>400.0</td>
      <td>2018</td>
      <td>Pocono_400</td>
      <td>Pocono Raceway</td>
    </tr>
    <tr>
      <th>4</th>
      <td>5</td>
      <td>17</td>
      <td>2</td>
      <td>Brad Keselowski</td>
      <td>Wurth (Roger Penske)</td>
      <td>Ford</td>
      <td>160</td>
      <td>running</td>
      <td>10</td>
      <td>37</td>
      <td>0</td>
      <td>160</td>
      <td>2.5</td>
      <td>paved track</td>
      <td>400.0</td>
      <td>2018</td>
      <td>Pocono_400</td>
      <td>Pocono Raceway</td>
    </tr>
  </tbody>
</table>
</div>



By converting each dataframe in the list to dicts, we can create a single dataframe.  The resulting dataframe columns will be a super set of each individual race dataframe's columns


```python
df = pd.DataFrame([row for r_df in race_data_frames for row in r_df.to_dict(orient='records')])
df.head()
```




<div class="dataframe-container">
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>#</th>
      <th>Car</th>
      <th>Driver</th>
      <th>Fin</th>
      <th>Laps</th>
      <th>Led</th>
      <th>Money</th>
      <th>PPts</th>
      <th>Pts</th>
      <th>Sponsor / Owner</th>
      <th>St</th>
      <th>Status</th>
      <th>race_length_laps</th>
      <th>race_length_miles</th>
      <th>race_name</th>
      <th>track_length_miles</th>
      <th>track_name</th>
      <th>track_type</th>
      <th>year</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>88</td>
      <td>Chevrolet</td>
      <td>Darrell Waltrip</td>
      <td>1</td>
      <td>119</td>
      <td>87</td>
      <td>21150.0</td>
      <td>NaN</td>
      <td>185.0</td>
      <td>Gatorade (DiGard Racing)</td>
      <td>4</td>
      <td>running</td>
      <td>119</td>
      <td>311.8</td>
      <td>Winston_Western_500</td>
      <td>2.62</td>
      <td>Riverside International Raceway</td>
      <td>road course</td>
      <td>1979</td>
    </tr>
    <tr>
      <th>1</th>
      <td>21</td>
      <td>Mercury</td>
      <td>David Pearson</td>
      <td>2</td>
      <td>119</td>
      <td>9</td>
      <td>14200.0</td>
      <td>NaN</td>
      <td>175.0</td>
      <td>Purolator (Wood Brothers)</td>
      <td>1</td>
      <td>running</td>
      <td>119</td>
      <td>311.8</td>
      <td>Winston_Western_500</td>
      <td>2.62</td>
      <td>Riverside International Raceway</td>
      <td>road course</td>
      <td>1979</td>
    </tr>
    <tr>
      <th>2</th>
      <td>11</td>
      <td>Oldsmobile</td>
      <td>Cale Yarborough</td>
      <td>3</td>
      <td>119</td>
      <td>3</td>
      <td>12675.0</td>
      <td>NaN</td>
      <td>170.0</td>
      <td>Busch (Junior Johnson)</td>
      <td>2</td>
      <td>running</td>
      <td>119</td>
      <td>311.8</td>
      <td>Winston_Western_500</td>
      <td>2.62</td>
      <td>Riverside International Raceway</td>
      <td>road course</td>
      <td>1979</td>
    </tr>
    <tr>
      <th>3</th>
      <td>73</td>
      <td>Oldsmobile</td>
      <td>Bill Schmitt</td>
      <td>4</td>
      <td>118</td>
      <td>0</td>
      <td>8000.0</td>
      <td>NaN</td>
      <td>160.0</td>
      <td>Old Milwaukee (Bill Schmitt)</td>
      <td>8</td>
      <td>running</td>
      <td>119</td>
      <td>311.8</td>
      <td>Winston_Western_500</td>
      <td>2.62</td>
      <td>Riverside International Raceway</td>
      <td>road course</td>
      <td>1979</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1</td>
      <td>Chevrolet</td>
      <td>Donnie Allison</td>
      <td>5</td>
      <td>118</td>
      <td>0</td>
      <td>7550.0</td>
      <td>NaN</td>
      <td>155.0</td>
      <td>Hawaiian Tropic (Hoss Ellington)</td>
      <td>17</td>
      <td>running</td>
      <td>119</td>
      <td>311.8</td>
      <td>Winston_Western_500</td>
      <td>2.62</td>
      <td>Riverside International Raceway</td>
      <td>road course</td>
      <td>1979</td>
    </tr>
  </tbody>
</table>
</div>



## Track Type
Currently, track type is not very descriptive


```python
df.track_type.unique()
```




    array(['road course', 'paved track'], dtype=object)



Using both the scraped track_type and track_length_miles, we'll classify a new track_type


```python
def track_type(row):
    if row['track_type'] == 'road course':
        return 'road course'
    elif row['track_length_miles'] >= 2.0:
        return 'superspeedway'
    elif row['track_length_miles'] >= 1.0:
        return 'intermediate'
    else:
        return 'short track'
```


```python
df['track_type'] = df.apply(track_type, axis=1)
df[['track_length_miles', 'track_type', 'track_name']].drop_duplicates()\
                                                      .sort_values('track_length_miles')\
                                                      .reset_index()\
                                                      .drop('index', axis=1)
```




<div class="dataframe-container">
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>track_length_miles</th>
      <th>track_type</th>
      <th>track_name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0.525</td>
      <td>short track</td>
      <td>Martinsville Speedway</td>
    </tr>
    <tr>
      <th>1</th>
      <td>0.526</td>
      <td>short track</td>
      <td>Martinsville Speedway</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0.533</td>
      <td>short track</td>
      <td>Bristol International Raceway</td>
    </tr>
    <tr>
      <th>3</th>
      <td>0.533</td>
      <td>short track</td>
      <td>Bristol Motor Speedway</td>
    </tr>
    <tr>
      <th>4</th>
      <td>0.533</td>
      <td>short track</td>
      <td>Bristol International Speedway</td>
    </tr>
    <tr>
      <th>5</th>
      <td>0.542</td>
      <td>short track</td>
      <td>Richmond Fairgrounds Raceway</td>
    </tr>
    <tr>
      <th>6</th>
      <td>0.596</td>
      <td>short track</td>
      <td>Nashville Speedway</td>
    </tr>
    <tr>
      <th>7</th>
      <td>0.625</td>
      <td>short track</td>
      <td>North Wilkesboro Speedway</td>
    </tr>
    <tr>
      <th>8</th>
      <td>0.750</td>
      <td>short track</td>
      <td>Richmond Raceway</td>
    </tr>
    <tr>
      <th>9</th>
      <td>0.750</td>
      <td>short track</td>
      <td>Richmond International Raceway</td>
    </tr>
    <tr>
      <th>10</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>Jeff Gordon Raceway</td>
    </tr>
    <tr>
      <th>11</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>Dover International Speedway</td>
    </tr>
    <tr>
      <th>12</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>Phoenix International Raceway</td>
    </tr>
    <tr>
      <th>13</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>ISM Raceway</td>
    </tr>
    <tr>
      <th>14</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>Dover Downs International Speedway</td>
    </tr>
    <tr>
      <th>15</th>
      <td>1.017</td>
      <td>intermediate</td>
      <td>North Carolina Motor Speedway</td>
    </tr>
    <tr>
      <th>16</th>
      <td>1.017</td>
      <td>intermediate</td>
      <td>North Carolina Speedway</td>
    </tr>
    <tr>
      <th>17</th>
      <td>1.058</td>
      <td>intermediate</td>
      <td>New Hampshire Motor Speedway</td>
    </tr>
    <tr>
      <th>18</th>
      <td>1.058</td>
      <td>intermediate</td>
      <td>New Hampshire International Speedway</td>
    </tr>
    <tr>
      <th>19</th>
      <td>1.366</td>
      <td>intermediate</td>
      <td>Darlington Raceway</td>
    </tr>
    <tr>
      <th>20</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Kentucky Speedway</td>
    </tr>
    <tr>
      <th>21</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Lowe's Motor Speedway</td>
    </tr>
    <tr>
      <th>22</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Kansas Speedway</td>
    </tr>
    <tr>
      <th>23</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Las Vegas Motor Speedway</td>
    </tr>
    <tr>
      <th>24</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Chicagoland Speedway</td>
    </tr>
    <tr>
      <th>25</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Texas Motor Speedway</td>
    </tr>
    <tr>
      <th>26</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Charlotte Motor Speedway</td>
    </tr>
    <tr>
      <th>27</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Homestead-Miami Speedway</td>
    </tr>
    <tr>
      <th>28</th>
      <td>1.522</td>
      <td>intermediate</td>
      <td>Atlanta International Raceway</td>
    </tr>
    <tr>
      <th>29</th>
      <td>1.522</td>
      <td>intermediate</td>
      <td>Atlanta Motor Speedway</td>
    </tr>
    <tr>
      <th>30</th>
      <td>1.540</td>
      <td>intermediate</td>
      <td>Atlanta Motor Speedway</td>
    </tr>
    <tr>
      <th>31</th>
      <td>1.949</td>
      <td>road course</td>
      <td>Sears Point Raceway</td>
    </tr>
    <tr>
      <th>32</th>
      <td>1.990</td>
      <td>road course</td>
      <td>Infineon Raceway</td>
    </tr>
    <tr>
      <th>33</th>
      <td>1.990</td>
      <td>road course</td>
      <td>Sonoma Raceway</td>
    </tr>
    <tr>
      <th>34</th>
      <td>1.990</td>
      <td>road course</td>
      <td>Sears Point Raceway</td>
    </tr>
    <tr>
      <th>35</th>
      <td>2.000</td>
      <td>road course</td>
      <td>Sears Point Raceway</td>
    </tr>
    <tr>
      <th>36</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>Auto Club Speedway</td>
    </tr>
    <tr>
      <th>37</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>Michigan International Speedway</td>
    </tr>
    <tr>
      <th>38</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>California Speedway</td>
    </tr>
    <tr>
      <th>39</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>Michigan Speedway</td>
    </tr>
    <tr>
      <th>40</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>Texas World Speedway</td>
    </tr>
    <tr>
      <th>41</th>
      <td>2.428</td>
      <td>road course</td>
      <td>Watkins Glen International</td>
    </tr>
    <tr>
      <th>42</th>
      <td>2.450</td>
      <td>road course</td>
      <td>Watkins Glen International</td>
    </tr>
    <tr>
      <th>43</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Pocono Raceway</td>
    </tr>
    <tr>
      <th>44</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Indianapolis Motor Speedway</td>
    </tr>
    <tr>
      <th>45</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Ontario Motor Speedway</td>
    </tr>
    <tr>
      <th>46</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Pocono International Raceway</td>
    </tr>
    <tr>
      <th>47</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Daytona International Speedway</td>
    </tr>
    <tr>
      <th>48</th>
      <td>2.520</td>
      <td>road course</td>
      <td>Sears Point Raceway</td>
    </tr>
    <tr>
      <th>49</th>
      <td>2.520</td>
      <td>road course</td>
      <td>Sears Point International Raceway</td>
    </tr>
    <tr>
      <th>50</th>
      <td>2.620</td>
      <td>road course</td>
      <td>Riverside International Raceway</td>
    </tr>
    <tr>
      <th>51</th>
      <td>2.660</td>
      <td>superspeedway</td>
      <td>Talladega Superspeedway</td>
    </tr>
    <tr>
      <th>52</th>
      <td>2.660</td>
      <td>superspeedway</td>
      <td>Alabama International Motor Speedway</td>
    </tr>
  </tbody>
</table>
</div>



## Wrapping Up
We now have single, tidy Pandas DataFrame with all Monster Energy NASCAR Cup Series results since 1979


```python
df.tail()
```




<div class="dataframe-container">
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>#</th>
      <th>Car</th>
      <th>Driver</th>
      <th>Fin</th>
      <th>Laps</th>
      <th>Led</th>
      <th>Money</th>
      <th>PPts</th>
      <th>Pts</th>
      <th>Sponsor / Owner</th>
      <th>St</th>
      <th>Status</th>
      <th>race_length_laps</th>
      <th>race_length_miles</th>
      <th>race_name</th>
      <th>track_length_miles</th>
      <th>track_name</th>
      <th>track_type</th>
      <th>year</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>52614</th>
      <td>99</td>
      <td>Chevrolet</td>
      <td>Derrike Cope</td>
      <td>34</td>
      <td>152</td>
      <td>0</td>
      <td>NaN</td>
      <td>0.0</td>
      <td>3.0</td>
      <td>StarCom Fiber (StarCom Racing)</td>
      <td>38</td>
      <td>running</td>
      <td>160</td>
      <td>400.0</td>
      <td>Pocono_400</td>
      <td>2.5</td>
      <td>Pocono Raceway</td>
      <td>superspeedway</td>
      <td>2018</td>
    </tr>
    <tr>
      <th>52615</th>
      <td>11</td>
      <td>Toyota</td>
      <td>Denny Hamlin</td>
      <td>35</td>
      <td>146</td>
      <td>0</td>
      <td>NaN</td>
      <td>0.0</td>
      <td>8.0</td>
      <td>FedEx Office (Joe Gibbs)</td>
      <td>10</td>
      <td>crash</td>
      <td>160</td>
      <td>400.0</td>
      <td>Pocono_400</td>
      <td>2.5</td>
      <td>Pocono Raceway</td>
      <td>superspeedway</td>
      <td>2018</td>
    </tr>
    <tr>
      <th>52616</th>
      <td>95</td>
      <td>Chevrolet</td>
      <td>Kasey Kahne</td>
      <td>36</td>
      <td>120</td>
      <td>0</td>
      <td>NaN</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>FDNY Foundation (Leavine Family Racing)</td>
      <td>22</td>
      <td>transmission</td>
      <td>160</td>
      <td>400.0</td>
      <td>Pocono_400</td>
      <td>2.5</td>
      <td>Pocono Raceway</td>
      <td>superspeedway</td>
      <td>2018</td>
    </tr>
    <tr>
      <th>52617</th>
      <td>32</td>
      <td>Ford</td>
      <td>Matt DiBenedetto</td>
      <td>37</td>
      <td>113</td>
      <td>0</td>
      <td>NaN</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>Zynga Poker (Archie St. Hilaire)</td>
      <td>32</td>
      <td>brakes</td>
      <td>160</td>
      <td>400.0</td>
      <td>Pocono_400</td>
      <td>2.5</td>
      <td>Pocono Raceway</td>
      <td>superspeedway</td>
      <td>2018</td>
    </tr>
    <tr>
      <th>52618</th>
      <td>43</td>
      <td>Chevrolet</td>
      <td>Bubba Wallace</td>
      <td>38</td>
      <td>108</td>
      <td>4</td>
      <td>NaN</td>
      <td>0.0</td>
      <td>1.0</td>
      <td>Weis Markets (Richard Petty Motorsports)</td>
      <td>19</td>
      <td>engine</td>
      <td>160</td>
      <td>400.0</td>
      <td>Pocono_400</td>
      <td>2.5</td>
      <td>Pocono Raceway</td>
      <td>superspeedway</td>
      <td>2018</td>
    </tr>
  </tbody>
</table>
</div>



In future posts, I'll revisit this data for further analysis.  For now, I'll save the DataFrame as a Python pickle file for easy ingestion


```python
df.to_pickle('race_details.pkl')
```
