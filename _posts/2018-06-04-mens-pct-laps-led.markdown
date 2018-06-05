---
layout: post
title:  "NASCAR Cup Series Percent Laps Led by Winners"
date:   2018-06-04 21:51:01 -0400
categories: jekyll update
---


```python
%matplotlib inline
import requests
from bs4 import BeautifulSoup
import re
import pandas as pd


plt.rcParams['figure.dpi'] = 300
BASE_URL = 'http://racing-reference.info'
```


```python
years = range(1979, 2019)

cup_results = [requests.get(BASE_URL + f'/raceyear/{year}/W') for year in years]
```

Check response codes


```python
set([r.status_code for r in cup_results])
```




    {200}



Each of these yearly cup series results pages has links to race details.  Lets find them all


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



For all of the extracted anchor tags, use the href attribute to request the race pages


```python
races = [requests.get(BASE_URL + a.attrs['href']) for a in race_anchors]
```


```python
set([r.status_code for r in races])
```




    {200}



## Extracting Race Results

To extract the race results stored as an html table, we can use the Pandas read_html function.

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



It would also be useful to extract the name of the track


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
df[['track_length_miles', 'track_type', 'track_name']].drop_duplicates().sort_values('track_length_miles')
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
      <th>277</th>
      <td>0.525</td>
      <td>short track</td>
      <td>Martinsville Speedway</td>
    </tr>
    <tr>
      <th>5698</th>
      <td>0.526</td>
      <td>short track</td>
      <td>Martinsville Speedway</td>
    </tr>
    <tr>
      <th>16744</th>
      <td>0.533</td>
      <td>short track</td>
      <td>Bristol International Raceway</td>
    </tr>
    <tr>
      <th>19259</th>
      <td>0.533</td>
      <td>short track</td>
      <td>Bristol Motor Speedway</td>
    </tr>
    <tr>
      <th>211</th>
      <td>0.533</td>
      <td>short track</td>
      <td>Bristol International Speedway</td>
    </tr>
    <tr>
      <th>111</th>
      <td>0.542</td>
      <td>short track</td>
      <td>Richmond Fairgrounds Raceway</td>
    </tr>
    <tr>
      <th>347</th>
      <td>0.596</td>
      <td>short track</td>
      <td>Nashville Speedway</td>
    </tr>
    <tr>
      <th>181</th>
      <td>0.625</td>
      <td>short track</td>
      <td>North Wilkesboro Speedway</td>
    </tr>
    <tr>
      <th>51649</th>
      <td>0.750</td>
      <td>short track</td>
      <td>Richmond Raceway</td>
    </tr>
    <tr>
      <th>10617</th>
      <td>0.750</td>
      <td>short track</td>
      <td>Richmond International Raceway</td>
    </tr>
    <tr>
      <th>49150</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>Jeff Gordon Raceway</td>
    </tr>
    <tr>
      <th>28081</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>Dover International Speedway</td>
    </tr>
    <tr>
      <th>10839</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>Phoenix International Raceway</td>
    </tr>
    <tr>
      <th>52199</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>ISM Raceway</td>
    </tr>
    <tr>
      <th>375</th>
      <td>1.000</td>
      <td>intermediate</td>
      <td>Dover Downs International Speedway</td>
    </tr>
    <tr>
      <th>76</th>
      <td>1.017</td>
      <td>intermediate</td>
      <td>North Carolina Motor Speedway</td>
    </tr>
    <tr>
      <th>21718</th>
      <td>1.017</td>
      <td>intermediate</td>
      <td>North Carolina Speedway</td>
    </tr>
    <tr>
      <th>37541</th>
      <td>1.058</td>
      <td>intermediate</td>
      <td>New Hampshire Motor Speedway</td>
    </tr>
    <tr>
      <th>15956</th>
      <td>1.058</td>
      <td>intermediate</td>
      <td>New Hampshire International Speedway</td>
    </tr>
    <tr>
      <th>241</th>
      <td>1.366</td>
      <td>intermediate</td>
      <td>Darlington Raceway</td>
    </tr>
    <tr>
      <th>42228</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Kentucky Speedway</td>
    </tr>
    <tr>
      <th>23567</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Lowe's Motor Speedway</td>
    </tr>
    <tr>
      <th>27179</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Kansas Speedway</td>
    </tr>
    <tr>
      <th>21761</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Las Vegas Motor Speedway</td>
    </tr>
    <tr>
      <th>26749</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Chicagoland Speedway</td>
    </tr>
    <tr>
      <th>20526</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Texas Motor Speedway</td>
    </tr>
    <tr>
      <th>406</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Charlotte Motor Speedway</td>
    </tr>
    <tr>
      <th>24470</th>
      <td>1.500</td>
      <td>intermediate</td>
      <td>Homestead-Miami Speedway</td>
    </tr>
    <tr>
      <th>141</th>
      <td>1.522</td>
      <td>intermediate</td>
      <td>Atlanta International Raceway</td>
    </tr>
    <tr>
      <th>13113</th>
      <td>1.522</td>
      <td>intermediate</td>
      <td>Atlanta Motor Speedway</td>
    </tr>
    <tr>
      <th>21632</th>
      <td>1.540</td>
      <td>intermediate</td>
      <td>Atlanta Motor Speedway</td>
    </tr>
    <tr>
      <th>22320</th>
      <td>1.949</td>
      <td>road course</td>
      <td>Sears Point Raceway</td>
    </tr>
    <tr>
      <th>28210</th>
      <td>1.990</td>
      <td>road course</td>
      <td>Infineon Raceway</td>
    </tr>
    <tr>
      <th>43690</th>
      <td>1.990</td>
      <td>road course</td>
      <td>Sonoma Raceway</td>
    </tr>
    <tr>
      <th>25201</th>
      <td>1.990</td>
      <td>road course</td>
      <td>Sears Point Raceway</td>
    </tr>
    <tr>
      <th>26663</th>
      <td>2.000</td>
      <td>road course</td>
      <td>Sears Point Raceway</td>
    </tr>
    <tr>
      <th>36896</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>Auto Club Speedway</td>
    </tr>
    <tr>
      <th>516</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>Michigan International Speedway</td>
    </tr>
    <tr>
      <th>20911</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>California Speedway</td>
    </tr>
    <tr>
      <th>19582</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>Michigan Speedway</td>
    </tr>
    <tr>
      <th>447</th>
      <td>2.000</td>
      <td>superspeedway</td>
      <td>Texas World Speedway</td>
    </tr>
    <tr>
      <th>8249</th>
      <td>2.428</td>
      <td>road course</td>
      <td>Watkins Glen International</td>
    </tr>
    <tr>
      <th>14925</th>
      <td>2.450</td>
      <td>road course</td>
      <td>Watkins Glen International</td>
    </tr>
    <tr>
      <th>19541</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Pocono Raceway</td>
    </tr>
    <tr>
      <th>17276</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Indianapolis Motor Speedway</td>
    </tr>
    <tr>
      <th>1048</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Ontario Motor Speedway</td>
    </tr>
    <tr>
      <th>623</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Pocono International Raceway</td>
    </tr>
    <tr>
      <th>35</th>
      <td>2.500</td>
      <td>superspeedway</td>
      <td>Daytona International Speedway</td>
    </tr>
    <tr>
      <th>16895</th>
      <td>2.520</td>
      <td>road course</td>
      <td>Sears Point Raceway</td>
    </tr>
    <tr>
      <th>11341</th>
      <td>2.520</td>
      <td>road course</td>
      <td>Sears Point International Raceway</td>
    </tr>
    <tr>
      <th>0</th>
      <td>2.620</td>
      <td>road course</td>
      <td>Riverside International Raceway</td>
    </tr>
    <tr>
      <th>11223</th>
      <td>2.660</td>
      <td>superspeedway</td>
      <td>Talladega Superspeedway</td>
    </tr>
    <tr>
      <th>307</th>
      <td>2.660</td>
      <td>superspeedway</td>
      <td>Alabama International Motor Speedway</td>
    </tr>
  </tbody>
</table>
</div>



## Percent Laps Led

Lets focus on making some calculations on percentage of laps led.

The race_length_laps column in our dataframe represents the scheduled number of laps in the race.  The Laps column represents the number of laps completed by each car.  Due to weather, darkness, or overtime finishes, races can finish at a number of laps different than the scheduled amount.  Lets broadcast the actual race length to all cars


```python
df_race_length = df.groupby(['year', 'race_name'])['Laps'].max()
df_race_length.head()
```




    year  race_name              
    1979  American_500               492
          Atlanta_500                328
          Busch_Nashville_420        420
          CRC_Chemicals_500          500
          CRC_Chemicals_Rebel_500    367
    Name: Laps, dtype: int64




```python
df = df.join(df_race_length, on=['year', 'race_name'], rsuffix='_total')
df[(df.year == 2017) & (df.race_name == 'Brickyard_400')][['#', 'Driver', 'race_length_laps', 'Laps', 'Laps_total']]
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
      <th>Driver</th>
      <th>race_length_laps</th>
      <th>Laps</th>
      <th>Laps_total</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>51415</th>
      <td>5</td>
      <td>Kasey Kahne</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51416</th>
      <td>2</td>
      <td>Brad Keselowski</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51417</th>
      <td>31</td>
      <td>Ryan Newman</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51418</th>
      <td>22</td>
      <td>Joey Logano</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51419</th>
      <td>20</td>
      <td>Matt Kenseth</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51420</th>
      <td>4</td>
      <td>Kevin Harvick</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51421</th>
      <td>19</td>
      <td>Daniel Suarez</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51422</th>
      <td>32</td>
      <td>Matt DiBenedetto</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51423</th>
      <td>37</td>
      <td>Chris Buescher</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51424</th>
      <td>47</td>
      <td>A.J. Allmendinger</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51425</th>
      <td>10</td>
      <td>Danica Patrick</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51426</th>
      <td>72</td>
      <td>Cole Whitt</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51427</th>
      <td>43</td>
      <td>Aric Almirola</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51428</th>
      <td>66</td>
      <td>Timmy Hill</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51429</th>
      <td>1</td>
      <td>Jamie McMurray</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51430</th>
      <td>27</td>
      <td>Paul Menard</td>
      <td>160</td>
      <td>167</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51431</th>
      <td>11</td>
      <td>Denny Hamlin</td>
      <td>160</td>
      <td>166</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51432</th>
      <td>95</td>
      <td>Michael McDowell</td>
      <td>160</td>
      <td>166</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51433</th>
      <td>13</td>
      <td>Ty Dillon</td>
      <td>160</td>
      <td>165</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51434</th>
      <td>6</td>
      <td>Trevor Bayne</td>
      <td>160</td>
      <td>162</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51435</th>
      <td>3</td>
      <td>Austin Dillon</td>
      <td>160</td>
      <td>162</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51436</th>
      <td>34</td>
      <td>Landon Cassill</td>
      <td>160</td>
      <td>162</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51437</th>
      <td>21</td>
      <td>Ryan Blaney</td>
      <td>160</td>
      <td>162</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51438</th>
      <td>55</td>
      <td>Gray Gaulding</td>
      <td>160</td>
      <td>162</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51439</th>
      <td>15</td>
      <td>Joey Gase</td>
      <td>160</td>
      <td>162</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51440</th>
      <td>33</td>
      <td>Jeffrey Earnhardt</td>
      <td>160</td>
      <td>162</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51441</th>
      <td>48</td>
      <td>Jimmie Johnson</td>
      <td>160</td>
      <td>158</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51442</th>
      <td>42</td>
      <td>Kyle Larson</td>
      <td>160</td>
      <td>154</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51443</th>
      <td>41</td>
      <td>Kurt Busch</td>
      <td>160</td>
      <td>149</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51444</th>
      <td>14</td>
      <td>Clint Bowyer</td>
      <td>160</td>
      <td>148</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51445</th>
      <td>77</td>
      <td>Erik Jones</td>
      <td>160</td>
      <td>148</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51446</th>
      <td>51</td>
      <td>B.J. McLeod</td>
      <td>160</td>
      <td>135</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51447</th>
      <td>78</td>
      <td>Martin Truex, Jr.</td>
      <td>160</td>
      <td>110</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51448</th>
      <td>18</td>
      <td>Kyle Busch</td>
      <td>160</td>
      <td>110</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51449</th>
      <td>17</td>
      <td>Ricky Stenhouse, Jr.</td>
      <td>160</td>
      <td>106</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51450</th>
      <td>88</td>
      <td>Dale Earnhardt, Jr.</td>
      <td>160</td>
      <td>76</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51451</th>
      <td>7</td>
      <td>J.J. Yeley</td>
      <td>160</td>
      <td>70</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51452</th>
      <td>38</td>
      <td>David Ragan</td>
      <td>160</td>
      <td>56</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51453</th>
      <td>24</td>
      <td>Chase Elliott</td>
      <td>160</td>
      <td>43</td>
      <td>167</td>
    </tr>
    <tr>
      <th>51454</th>
      <td>23</td>
      <td>Corey LaJoie</td>
      <td>160</td>
      <td>9</td>
      <td>167</td>
    </tr>
  </tbody>
</table>
</div>



Now we can calculate the percentage of the race led


```python
df['pct_led'] = df['Led'] / df['Laps_total']
df[['#', 'Led', 'Laps_total', 'pct_led']].head()
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
      <th>Led</th>
      <th>Laps_total</th>
      <th>pct_led</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>88</td>
      <td>87</td>
      <td>119</td>
      <td>0.731092</td>
    </tr>
    <tr>
      <th>1</th>
      <td>21</td>
      <td>9</td>
      <td>119</td>
      <td>0.075630</td>
    </tr>
    <tr>
      <th>2</th>
      <td>11</td>
      <td>3</td>
      <td>119</td>
      <td>0.025210</td>
    </tr>
    <tr>
      <th>3</th>
      <td>73</td>
      <td>0</td>
      <td>119</td>
      <td>0.000000</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1</td>
      <td>0</td>
      <td>119</td>
      <td>0.000000</td>
    </tr>
  </tbody>
</table>
</div>



By filtering the dataframe for only winners, we can calculate the percentage of total laps led by all the winners in a given year


```python
df_winners = df[df['Fin'] == 1].copy()
df_pct_led_overall = df_winners.groupby('year').apply(lambda x: pd.Series({'pct_led_overall': x['Led'].sum() / x['Laps_total'].sum()}))
```

Now we can plot both the race percentages and the yearly percentage on the same plot


```python
ax = df_winners.plot(x='year',
                     y='pct_led',
                     kind='scatter',
                     title='MENCS Modern Era Percent Laps Led by the Winner',
                     figsize=(16, 9),
                     c='white',
                     marker='.')
df_pct_led_overall.reset_index().plot('year', 'pct_led_overall', ax=ax, c='#7dbe32', lw=2.5)
ax.set_facecolor("black")
ax.text(0.95, 0.01, '©jasondamiani.com',
        verticalalignment='bottom', horizontalalignment='right',
        transform=ax.transAxes,
        color='white', fontsize=8);
```


![png](/images/output_43_0.png)
