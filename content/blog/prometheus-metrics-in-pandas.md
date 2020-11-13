+++
author = "Ricardo Rocha"
date = "2020-11-13T22:00:00+00:00"
description = "An example for GPU workload metrics"
tags = ["prometheus", "pandas", "jupyter"]
title = "Analysing Prometheus Metrics in Pandas"
+++

I spend more and more time working with [Prometheus](https://prometheus.io/) as it's the default monitoring / metric collection system for Kubernetes clusters at work and other projects i have. It's also a great tool overall i wish i had for longer. Pair it with [Grafana](https://grafana.com/) and it gives a well integrated monitoring solution:
{{< figure src="/images/blog/prometheus-cluster.png"
    title="" width="100%" >}}

While a lot can be done with the timeseries data using PromQL and Grafana, i often miss Pandas to easily reorganize the data. Looking around i found [prometheus-pandas](https://github.com/dcoles/prometheus-pandas), a very simple python library that does the job to convert Promethes metrics into Pandas dataframes.

#### Query Prometheus

In this post i'll use a slightly modified code based on that library to convert the metrics to dataframes, aggregate and reshape the data and finally plot it. Our query's result has two dimensions (cloud and gpu) and includes metrics collected every 15s.

```python
import requests
from urllib.parse import urljoin

api_url = "http://localhost:1111"

# Base functions querying prometheus
def _do_query(path, params):
    resp = requests.get(urljoin(api_url, path), params=params)
    if not (resp.status_code // 100 == 200 or resp.status_code in [400, 422, 503]):
        resp.raise_for_status()

    response = resp.json()
    if response['status'] != 'success':
        raise RuntimeError('{errorType}: {error}'.format_map(response))

    return response['data']

# Range query
def query_range(query, start, end, step, timeout=None):
    params = {'query': query, 'start': start, 'end': end, 'step': step}
    params.update({'timeout': timeout} if timeout is not None else {})

    return _do_query('api/v1/query_range', params)

# Perform our query
data = query_range(
    'sum by(cloud, gpu) (duration{workload="fitting"})',
    '2020-11-12T11:15:39Z', '2020-11-12T12:19:10Z', '15s')
data
```

Inspecting the results we can see the json format returned.
```yaml
    {'resultType': 'matrix',
     'result': [{'metric': {'cloud': 'aws', 'gpu': 'k80'},
       'values': [[1605180639, '4.102823257446289'],
        [1605180654, '4.102823257446289'],
        ...
      {'metric': {'cloud': 'google', 'gpu': 'v100'},
       'values': [[1605181374, '0.8236739635467529'],
        ...
        [1605183234, '0.8240261077880859']]}]}
```

This format is verbose but very easy to handle in Python.

#### Pandas Dataframes

From here to a Pandas dataframe only takes a couple lines. [prometheus-pandas](https://github.com/dcoles/prometheus-pandas) supports scalar, series and dataframes but in this case we need to build a MultiIndex, so here's a slight variation of the function available in the library.

```python
import numpy as np
import pandas as pd

# Prometheus range query giving a Pandas dataframe as output
df = pd.DataFrame({
        (r['metric']['cloud'], r['metric']['gpu']):
             pd.Series((np.float64(v[1]) for v in r['values']),
                    index=(pd.Timestamp(v[0], unit='s') for v in r['values']))
        for r in data['result']})
df.head(1)
```

<!DOCTYPE HTML>
<html>
<div>
<style  type="text/css" >
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead tr th {
        text-align: left;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th colspan="4" halign="left">aws</th>
      <th colspan="4" halign="left">azure</th>
      <th colspan="2" halign="left">cern</th>
      <th colspan="5" halign="left">google</th>
    </tr>
    <tr>
      <th></th>
      <th>k80</th>
      <th>m60</th>
      <th>t4</th>
      <th>v100</th>
      <th>k80</th>
      <th>m60</th>
      <th>p100</th>
      <th>p4</th>
      <th>t4</th>
      <th>v100</th>
      <th>k80</th>
      <th>p100</th>
      <th>p4</th>
      <th>t4</th>
      <th>v100</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>11:15:39</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>2.1</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>4.0</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>
</html>

Let's take this timeseries and calculate the mean still using our
MultiIndex from above.
```python
df2 = df.mean()
df2
```
```yaml
aws     k80     4.081384
        m60     3.011447
        t4      2.046543
        v100    0.823264
azure   k80     4.076168
        m60     2.182651
        p100    0.792160
        p4      1.626118
cern    t4      2.034331
        v100    0.675767
google  k80     4.103761
        p100    0.816900
        p4      2.721993
        t4      2.059852
        v100    0.823929
dtype: float64
```

#### Shuffling Data

The next and last step is to reorganize the data so we can plot clouds vs gpu
cards. To do that we need to pivot (unstack) our hierarchical index - for the
table view we can unstack with 1 level, and we'll also highlight the best
result per cloud (in this case the best result being the minimum value).

```python
# Unstack to plot clouds against gpus, and 
df3 = df2.unstack(level=1)
df3[['m60', 'p4', 't4', 'k80', 'p100', 'v100']].style.highlight_min(
    color='lightblue', axis=1).format("{:.2e}", na_rep="n/a")
```
<!DOCTYPE HTML>
<html>
<div>
<style  type="text/css" >
#T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow0_col5,#T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow1_col4,#T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow2_col5,#T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow3_col4{
            background-color:  lightblue;
        }</style><table id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cfe" ><thead>    <tr>        <th class="blank level0" ></th>        <th class="col_heading level0 col0" >m60</th>        <th class="col_heading level0 col1" >p4</th>        <th class="col_heading level0 col2" >t4</th>        <th class="col_heading level0 col3" >k80</th>        <th class="col_heading level0 col4" >p100</th>        <th class="col_heading level0 col5" >v100</th>    </tr></thead><tbody>
                <tr>
                        <th id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cfelevel0_row0" class="row_heading level0 row0" >aws</th>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow0_col0" class="data row0 col0" >3.01e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow0_col1" class="data row0 col1" >n/a</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow0_col2" class="data row0 col2" >2.05e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow0_col3" class="data row0 col3" >4.08e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow0_col4" class="data row0 col4" >n/a</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow0_col5" class="data row0 col5" >8.23e-01</td>
            </tr>
            <tr>
                        <th id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cfelevel0_row1" class="row_heading level0 row1" >azure</th>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow1_col0" class="data row1 col0" >2.18e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow1_col1" class="data row1 col1" >1.63e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow1_col2" class="data row1 col2" >n/a</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow1_col3" class="data row1 col3" >4.08e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow1_col4" class="data row1 col4" >7.92e-01</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow1_col5" class="data row1 col5" >n/a</td>
            </tr>
            <tr>
                        <th id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cfelevel0_row2" class="row_heading level0 row2" >cern</th>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow2_col0" class="data row2 col0" >n/a</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow2_col1" class="data row2 col1" >n/a</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow2_col2" class="data row2 col2" >2.03e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow2_col3" class="data row2 col3" >n/a</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow2_col4" class="data row2 col4" >n/a</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow2_col5" class="data row2 col5" >6.76e-01</td>
            </tr>
            <tr>
                        <th id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cfelevel0_row3" class="row_heading level0 row3" >google</th>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow3_col0" class="data row3 col0" >n/a</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow3_col1" class="data row3 col1" >2.72e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow3_col2" class="data row3 col2" >2.06e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow3_col3" class="data row3 col3" >4.10e+00</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow3_col4" class="data row3 col4" >8.17e-01</td>
                        <td id="T_83bb0758_25fe_11eb_a5ef_f14cb32a3cferow3_col5" class="data row3 col5" >8.24e-01</td>
            </tr>
    </tbody></table>
</html>

And just as easily we can plot the results, just setting levels to 0 when
unstacking.

```python
# Plot
fig, ax = plt.subplots(figsize=(10, 5))
plt.legend(loc=2, fontsize='x-small')

ax.set_ylabel('Approximate Fitting Duration (seconds)')
ax.set_xlabel('GPU Cards')
df3 = df2.unstack(level=0)
df3.plot.bar(rot=0, ax=ax)
```

![png](/images/blog/pandas_12_2.png)

Plotting data for a single cloud is similarly easy.

```python
fig, ax = plt.subplots(figsize=(10, 5))

ax.set_ylabel('Approximate Fitting Duration (seconds)')
ax.set_xlabel('Google GPU Cards')
_ = df3['google'].plot.bar(rot=0, ax=ax)
```

![png](/images/blog/pandas_13_0.png)

There are likely cases where the conversion is trickier, but even with more
than one dimension it is possible (and very useful) to handle the data with
Pandas and dataframes.
