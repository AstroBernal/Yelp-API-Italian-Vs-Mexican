# Italian vs. Mexican Food

### Using the Yelp API and US Census data

---

The below script provides an analytic approach for assessing American preferences of Italian vs. Mexican food. Using data from the US Census and the Yelp API, the script randomly selects over 500 zip codes and aggregates Yelp reviews from the 20 most popular Italian and Mexican restaurants in each area. The data is then parsed and analyzed using Python Pandas and Matplotlib.


```python
# Dependencies
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import requests
import time
import json
import seaborn
from scipy.stats import ttest_ind

# Yelp API Key (Keys Hidden)
ykey_id = "XXX"
ykey_secret = "XXX"
ykey_access_token = "gl6k6JmewUhzjMVBv0I2x4Bz_NRiEggSqjlGbTaejmbzvBJXgI36FPgWoqBnEL9QQ6wU5H4h41dxPkxVjHFlawtH69m1kcXQuHev5PuWBtcdBEAbdJR0HNl3d4tpWXYx"
```

## Zip Code Sampling


```python
# Import the census data into a Pandas DataFrame
census_pd = pd.read_csv("Census_Data.csv")

# Preview the data
census_pd.head()
```

![](assets/01_Census.png)


```python
# Select all zip codes with populations over 1000 from a pre-set list of 700 randomly selected zip code locations 
selected_zips = census_pd.sample(n=700)
selected_zips = selected_zips[selected_zips["Population"].astype(int) > 1000]

# Visualize
selected_zips.head()
```

![](assets/02_Zips_Population_greater_1000.png)


```python
# Show the total number of zip codes that met our population cut-off
selected_zips.count()
```


    Zipcode              524
    Address              524
    Population           524
    Median Age           524
    Household Income     523
    Per Capita Income    524
    dtype: int64


```python
# Show the average population of our representive sample set
selected_zips["Population"].mean()
```


    13949.314885496184


```python
# Show the average population of our representive sample set
selected_zips["Household Income"].mean()
```


    56315.009560229446


```python
# Show the average population of our representive sample set
selected_zips["Median Age"].mean()
```


    39.937977099236626



## Yelp Data Retrieval


```python
# Create Two DataFrames to store the Italian andMexican Data 
italian_data = pd.DataFrame();
mexican_data = pd.DataFrame();

# Setup the DataFrames to have appropriate columns
italian_data["Zip Code"] = ""
italian_data["Italian Review Count"] = ""
italian_data["Italian Average Rating"] = ""
italian_data["Italian Weighted Rating"] = ""

mexican_data["Zip Code"] = ""
mexican_data["Mexican Review Count"] = ""
mexican_data["Mexican Average Rating"] = ""
mexican_data["Mexican Weighted Rating"] = ""

# Include Yelp Token
headers = {"Authorization": "Bearer gl6k6JmewUhzjMVBv0I2x4Bz_NRiEggSqjlGbTaejmbzvBJXgI36FPgWoqBnEL9QQ6wU5H4h41dxPkxVjHFlawtH69m1kcXQuHev5PuWBtcdBEAbdJR0HNl3d4tpWXYx"}
counter = 0

# Loop through every zip code
for index, row in selected_zips.iterrows():
    
    # Add to counter
    counter = counter + 1
    
    # Create two endpoint URLs:
    target_url_italian = "https://api.yelp.com/v3/businesses/search?term=Italian&location=%s" % (row["Zipcode"])
    target_url_mexican = "https://api.yelp.com/v3/businesses/search?term=Mexican&location=%s" % (row["Zipcode"])
    
    # Print the URLs to ensure logging
    print(counter)
    print(target_url_italian)
    print(target_url_mexican)
    
    # Get the Yelp Reviews
    yelp_reviews_italian = requests.get(target_url_italian, headers=headers).json()
    yelp_reviews_mexican = requests.get(target_url_mexican, headers=headers).json()
    
    # Calculate the total reviews and weighted rankings
    italian_review_count = 0
    italian_weighted_review = 0
    mexican_review_count = 0
    mexican_weighted_review = 0
    
    # Use Try-Except to handle errors
    try:
        
        # Loop through all records to calculate the review count and weighted review value
        for business in yelp_reviews_italian["businesses"]:

            italian_review_count = italian_review_count + business["review_count"]
            italian_weighted_review = italian_weighted_review + business["review_count"] * business["rating"]

        for business in yelp_reviews_mexican["businesses"]:
            mexican_review_count = mexican_review_count + business["review_count"]
            mexican_weighted_review = mexican_weighted_review + business["review_count"] * business["rating"] 
        
        # Append the data to the appropriate column of the data frames
        italian_data.set_value(index, "Zip Code", row["Zipcode"])
        italian_data.set_value(index, "Italian Review Count", italian_review_count)
        italian_data.set_value(index, "Italian Average Rating", italian_weighted_review / italian_review_count)
        italian_data.set_value(index, "Italian Weighted Rating", italian_weighted_review)

        mexican_data.set_value(index, "Zip Code", row["Zipcode"])
        mexican_data.set_value(index, "Mexican Review Count", mexican_review_count)
        mexican_data.set_value(index, "Mexican Average Rating", mexican_weighted_review / mexican_review_count)
        mexican_data.set_value(index, "Mexican Weighted Rating", mexican_weighted_review)

    except:
        print("Uh oh")
        
```

![](assets/03_Yelp_API_results.png)

```python
# Preview Italian Data
italian_data.head()
```

![](assets/04_Italian_sample.png)


```python
# Preview Mexican Data
mexican_data.head()
```

![](assets/05_Mexican_Sample.png)

## Calculate Summaries


```python
# Total Mexican Reviews
mexican_data["Mexican Review Count"].sum()
```


    476889


```python
# Total Italian Reviews
italian_data["Italian Review Count"].sum()
```


    573733


```python
# Average Mexican Rating
mexican_data["Mexican Weighted Rating"].sum() / mexican_data["Mexican Review Count"].sum()
```


    3.909732663156416


```python
# Average Italian Rating
italian_data["Italian Weighted Rating"].sum() / italian_data["Italian Review Count"].sum()
```


    3.9446641556263975


```python
# Combine DataFrames into a single DataFrame
combined_data = pd.merge(mexican_data, italian_data, on="Zip Code")
combined_data.head()
```

![](assets/06_Combined_Data.png)


```python
# Determine Total Review Count and Rating "Wins" by City (Winner Take All)
combined_data["Rating Wins"] = np.where(combined_data["Mexican Average Rating"] > combined_data["Italian Average Rating"], "Mexican", "Italian")
combined_data["Review Count Wins"] = np.where(combined_data["Mexican Review Count"] > combined_data["Italian Review Count"], "Mexican", "Italian")
```


```python
# View Combined Data
combined_data.head()
```

![](assets/07_Combined_Data.png)


```python
# Tally number of cities where one food variety wins on ratings over the other
combined_data["Rating Wins"].value_counts()
```


    Mexican    273
    Italian    245
    Name: Rating Wins, dtype: int64


```python
# Tally number of cities where one food variety wins on review counts over the other
combined_data["Review Count Wins"].value_counts()
```


    Italian    298
    Mexican    220
    Name: Review Count Wins, dtype: int64



## Display Summary of Results


```python
# Model 1: Head-to-Head Review Counts
italian_summary = pd.DataFrame({"Review Counts": italian_data["Italian Review Count"].sum(),
                                "Rating Average": italian_data["Italian Average Rating"].mean(),
                                "Review Count Wins": combined_data["Review Count Wins"].value_counts()["Italian"],
                                "Rating Wins": combined_data["Rating Wins"].value_counts()["Italian"]}, index=["Italian"])

mexican_summary = pd.DataFrame({"Review Counts": mexican_data["Mexican Review Count"].sum(),
                                "Rating Average": mexican_data["Mexican Average Rating"].mean(),
                                "Review Count Wins": combined_data["Review Count Wins"].value_counts()["Mexican"],
                                "Rating Wins": combined_data["Rating Wins"].value_counts()["Mexican"]}, index=["Mexican"])

final_summary = pd.concat([mexican_summary, italian_summary])
final_summary
```

![](assets/08_Final_Summary.png)


```python
# Plot Rating Average
plt.clf()
final_summary["Rating Average"].plot.bar()
plt.title("Yelp Ratings by Food Variety")
plt.ylabel("Average Rating")
plt.xlabel("Food Variety")
plt.xticks(rotation="horizontal")
plt.grid(True)
plt.show()
```


![png](output_25_0.png)



```python
# Plot Rating Wins
plt.clf()
final_summary["Rating Wins"].plot.bar()
plt.title("# of Zip Codes with Preference by Food Variety According to Rating")
plt.ylabel("Number of Zip Codes")
plt.xlabel("Food Variety")
plt.xticks(rotation="horizontal")
plt.grid(True)
plt.show()
```


![png](output_26_0.png)



```python
# Plot Review Count
plt.clf()
final_summary["Review Counts"].plot.bar()
plt.title("Yelp Review Counts by Food Variety")
plt.ylabel("Reviwe Counts")
plt.xlabel("Food Variety")
plt.xticks(rotation="horizontal")
plt.grid(True)
plt.show()
```


![png](output_27_0.png)



```python
# Plot Review Count
plt.clf()
final_summary["Review Count Wins"].plot.bar()
plt.title("# of Zip Codes with Preference by Food Variety According to Review Counts")
plt.ylabel("Number of Zip Codes")
plt.xlabel("Food Variety")
plt.xticks(rotation="horizontal")
plt.grid(True)
plt.show()
```


![png](output_28_0.png)



```python
# Histogram Italian Food (Ratings)
plt.figure()

# Subplot 1 (Italian)
plt.subplot(121)
combined_data["Italian Average Rating"].plot.hist(bins=[0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5, 4, 4.5, 5.0], color="blue", alpha=0.6)
plt.xlabel("Italian Restaurant Ratings")
plt.xlim([1, 5.0])
plt.ylim([0, 400])

# Subplot 2 (Mexican)
plt.subplot(122)
combined_data["Mexican Average Rating"].plot.hist(bins=[0, 0.5, 1, 1.5, 2, 2.5, 3, 3.5, 4, 4.5, 5.0], color="red", alpha=0.6)
plt.xlabel("Mexican Retaurant Ratings")
plt.xlim([1, 5.0])
plt.ylim([0, 400])

# Show Plot
plt.show()
```


![png](output_29_0.png)


## Statistical Analysis


```python
# Run a t-test on average rating and number of reviewers
mexican_ratings = combined_data["Mexican Average Rating"]
italian_ratings = combined_data["Italian Average Rating"]

mexican_review_counts = combined_data["Mexican Review Count"]
italian_review_counts = combined_data["Italian Review Count"]
```


```python
# Run T-Test on Ratings
ttest_ind(mexican_ratings.values, italian_ratings.values, equal_var=False)
```




    Ttest_indResult(statistic=1.0702777109866186, pvalue=0.28474610492878616)




```python
# Run T-Test on Review Counts
ttest_ind(mexican_review_counts.values, italian_review_counts.values, equal_var=False)
```




    Ttest_indResult(statistic=-1.9031271166792225, pvalue=0.057326353522100949)



## Conclusions
---
Based on our analysis, it is clear that the American preference for Italian and Mexican food are similar in nature. As a whole, Americans rate Mexican and Italian restaurants at statistically similar scores (Avg score: 3.8, p-value: 0.285). However, there  exists statistically significant evidence that Americans write more reviews of Italian restaurants than Mexican. This may indicate that there is an increased interest in visiting Italian restaurants at an experiential level. However, it may also merely suggest that Yelp users enjoy writing reviews on Italian restaurants more than Mexican restaurants.
