# Weather Got You Down?
### Predicting Depression Trends Using High-Resolution Weather Data


## Google Trends Data Collection and Processing

To collect data, we utilized `R` and the `R` library `gtrendsR`. This allowed us to avoid manually querying Google Trends through a browser. More importantly using `gtrendsR` massively simplified the following steps of scaling, concatenation, and smoothing as the data was returned in an `R` dataframe.


### Google Trends Data Scaling and Concatenation

We quickly realized that Google Trends only provides daily data if the user is requesting 3 months worth of data or less. This became quickly problematic since we were interesting in looking at years of Trends data. To further complicate things, Google Trends data are automatically normalized so that the greatest point within a given query is set to 100. Therefore, multiple queries are not directly comparable and must be rescaled relative to each other to reach any meaningful conclusions. 

The `get_dates_list()` function in `helpers.R` splits a user’s date range into user-specified intervals with user-specified overlap. In our case, we wanted to split 10 years into 90-day intervals with 1 month of overlap (to ensure that we’d be able to rescale the data accurately, as described below). This is necessary because each `gtrendsR` query must have it’s own 90-day date interval in order to grab daily Google Trends data. 

Then, the `concat_trends()` function in `helpers.R` accepts a list of lists of date intervals (exactly in the format generated by `get_dates_list()`) as well as user-specified search terms, and locations. Now, we are ready to start querying for Google Trends data.

We begin by querying twice for data, using the first two overlapping date intervals (for instance, we might query for data from `January 1st, 2010` to `March 31st, 2010` and from `March 1st, 2010` to `May 31st, 2010`). Then, we find the mean number of hits (normalized by Google to be from 0 to 100) for each set of data in the overlapping date interval. Then, we rescale dataset with the larger of the two means so that its mean is the same as that of the dataset with the smaller of the two means. Then the two datasets are concatenated (with the average of the two datasets hits used in the overlapping region) to form a single dataset. 

This process is then iterated (but now between the old, growing dataset and a fresh query) until the end of the specified date range has been reached, resulting in a single dataset with a now continuous values for Google hits. 

### The Robin Williams Effect

Notably, there is an extremely disproportionate spike in Google searches for “depression” the day following Robin Williams death. Presumably, this is because Robin Williams’ death, combined with his depression, massively publicized depression in the US, and not because more people were actually depressed. 

Thus, since we were using depression searches as a proxy for depression rates, `concat_trends()` contains an option to correct for this effect by replacing the hits for the dates within a 7-day radius of Robin Williams’ death with a 7-day rolling average of hits, resulting in a reasonably smooth curve in that area, sans “Robin Williams Effect”. 


### Google Trends Data Smoothing

Naturally, this day-to-day data is quite variable, and this caused some issued for the accuracy of our model. To remove some of the noise, `normalize_trends.R` runs a rolling average (window of 7 days, centered on the middle day, weighted roughly as a normal distribution) on the raw data and returns the smoothed data. 


---

In the end, using this method gave us more accurate, higher-resolution data than if we had simply used the monthly data that would have been provided had we queried for all 10 years at once. 


## Feature Engineering

Features that are deemed to be related to depression and mood are extracted from the csv into a panda dataframe. Most features are continuous data (temperature, wind speed, suntime, etc.). However, some data are categorical data such as weather type. This was much more difficult to deal with as we have to transform the weather type code into a one hot encoding that binarizes the categories. 


## Neural Network Implementation

The Keras library was used to construct the neural network with 2 hidden layers (each with 10 neurons). Other architectures (relu activation function, MSE loss function, adam optimizer, and 20% validation split) were chosen by experimenting with various parameters and determining the optimum. To improve upon the performance of the model, we have also implemented batch normalization that normalizes the data before it is passed on to the next layer in the neural network. This ensures that we don't have any vanishing gradient problem where the activation function in the neural network does not learn anything and therefore don't improve.


## Training and Testing

8 years worth of data were used to train the neural network, while 2 years are reserved as a blind test. When training, 80% of the data are used to train and 20% are reserved as validation data. The validation data is used to prevent overfitting model and ensure that our model trained on that 80% of the data is generalizable.
