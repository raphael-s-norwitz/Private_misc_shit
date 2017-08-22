# README for Domain Spoofing Code:

## Queries:

### impression_over_uuid.hql

This query groups impression/click data by partnerid and property. It then returns the number of impressions / the number of uuids, and is used by parse_features.py 

### device_type.hql

This query groups impression/click data by device type. It is used to generate the device type feature in parse_features.py 

### country_cross_state.hql

This query groups by location by hashing concatenations of country and state. It is used to generate the geo feature in parse_features.py. 

### all_features.hql:

NOTE: this query isn't used by parse_features.py, but I was trying to adapt the code so that it would.

This is a simple hive query returning all the features for my main fraud model in parse_features.py

It groups impression/click data in adimpclickevents2 in hive by:

- partnerid
- propertyid
- country
- state
- reqadwidth and reqadheight <ad size>
- pfc

The output schema is:

```<partnerid> <propertyid> <# impressions> <# clicks> <# different locations> <# different ad sizes> <platform code> <# different uuids> <# tpuuids> <# ip addresses>```


## Python scripts

## parse_features.py

This is the most important file. It takes in data from all_features.hql and returns the output of a grid-search of different outlier vectors for a DBSCAN clustering model

NOTE: to use you must define a property list. I removed my property list definition.

at the top of main add a line like the following:

properties = ["msn.com", "foxnews.com", "weather.com"];

Except with the properties you want to cluster.

The model clusters individual supply side partners FOR A GIVEN PROPERTY based on the following 3 features:

- # of distinct uuids (cookies) / # of impressions
_ K-L divergence of device-type histograms
- K-L divergence of location histograms

For each property, the histogram only includes device-types, locations...etc seen for that property only

Run as:

```python2 parse_all.py <path to data root> <impression / uuid file name> <device data file name> <geo location file name>```

This script is designed for custom features to be added to two different lists.

There is a 'feature-ssp' vector (cluster_by in the code)

Each feature should also have  

NOTE: runs for me with python2. I have had some problems with python3, or the default python version on my drawbridge machine


#### Key Functions:

`get_file_fields`:

This function just parses a CSV file and returns a 2-dimensional list
- each row is a line in the file
- the columns are the different words in each line

args:
    path to file as a string
returns:
    python list (2-dimensional)

`get_indexed_file_mat`:

This function takes in a 2-dimensional python array (from get_file_fields) and returns a dictionary with keys and values for each row made up of keys in the file.

The dictionary maps tuples of 'identifying' elements to lists of 'value' rows.

The 'identifying' elements for this model are SSP and property (since we are clustering SSP cross property for different properties).

The 'value' features are different for each feature in the model. In short they are the comma separated values in the rows which you actually want to feed into the model.

args:
    - `mat`: 2-dimensional list (a parsed CSV file
    - `iden`: the indexes of the 'identifying' features for each row
    - `vals`: the indexes of the 'value' features for each row
returns:
    - `ledger`: a python dictionary mapping identifying tuples to lists. The lists correspond to different rows of the CSV, and contain lists of relevant values for the model

The optional parameters keys_idx and keys allow the user to define 'key identifiers' (in the case of our model, properties like cnn/foxnews..etc)

The function will only return rows of the CSV whose 'key identifiers', match those defined in 'keys' (in our case only properties, so a vector of domains)

(when I was running this, I used them to remove all properties but msn.com, foxnews.com and weather.com from the dataset).

Basically remove the last two parameters and it will cluster all combinations of ssp and properties in the dataset.

`partition_by_key_feature`:

This function partitions the output of get_indexed_file_mat into smaller dictionaries containing only rows of the original dictionary which match the key-property. 

args:
    - `mappying`: (python dictionary) a  mapping tuples of 'identifying features' to lists of lists, each sublist containing values which will be transformed into a feature value.
    - `partitions`: (list) values to 'partition' the dataset by. In our case the feature to partition by is the SSP, and in all the queries I've written, the SSP index is 0.
    - `subpartition_idx`: (int) index of the 'key' in mapping which should be evaluated when partitioning. (defaults to 0 see line above)

returns:
    - `ret_mapping`: (python dictionary) This dictionary maps partitions names to dictionaries with the set of <key, value> pairs in 'mapping' whose partitions match the outer dictionary's partition key. 
    - `ret_mapping_keys`: (python dictionary) maps partition names (i.e. 'partitions' is the set of keys) to lists of sub_values for that partition. We are using this to map properties to lists of SSPs who serve their impressions

`consolodate_discrete_count`:

This function generates histogram features from the list of list of list returned by partition_by_key_feature

`combine_lists_nested`:

combine property SSP lists. Pretty straightforward

`grid_cluser_search_rel`:

This performs a grid search of DBSCANs, the information from which is then printed by main.

#### Overview of main:

The aforementioned functions are used to populate two lists, namely cluster_by and and features.

cluster_by is a list of `ret_mapping_keys` returned from partition_by_key_features.

features is a list of lists. The first element is a `ret_mapping`, and the second is a 1 or 0.

If it is a 0, the feature is evaluated as a raw value. If it is a 1, the feature is evaluated like a histogram using the K-L divergence. 

### parse_all.py:

This script is to get and compare CTRs from different partners over different ranges of time

run as: 

```python parse_all.py <path to data directory>```

The data directory should contain 'partner' subdirectories, with each subdirectory containing files, each with an 'interval' worth of data.

The interval is the time over which the CTR is to be calculated. Say the intervals are all days. the tree should look like this:

<root>
    |_ partner_1
        |_day_1
        |_day_2
        ... 
        |_day_n
    |_ partner_2
        |_day_1
        |_day_2
        ... 
        |_day_n
    |_ ...
    |_ partner_n
        |_day_1
        |_day_2
        ... 
        |_day_n
     
The script will then print out the CTRs of the different partners as well as the top CTRs seen, along with other information.

