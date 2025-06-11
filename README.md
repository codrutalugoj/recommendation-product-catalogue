# Product Deduplication / Entity Resolution for Technological Products

## Data Exploration
- displaying general info on the dataframe
- checking data types 
- counting missing values in columns
- naive check for exact duplicated rows
- checking that numeric columns in bd_technologies have positive values.

## Data Cleaning

1. missing values
The previous step revealed that 1st dataset (ts_technologies) has a large number of missing values for url and description. For now I use a placeholder string "UNKNOWN" instead of leaving a NaN or using an empty string (because an empty string might give an exact match in a future text similarity comparison).
The idea was that I could discount the weight of the placeholder word if I were to use e.g. TD-IDF model. 
I handled all missing values this way.

2. renamed columns in the 2nd dataset (bd_technologies) that represent the same information in the 2 datasets:
"product_name": "name", "seller_website": "url", "main_category": "parent_category", "categories": "category"

3. removed square brackets and quotes from "categories" in bd_technologies
4. dropped columns that we can't use for entity matching because there was no equivalent in the other dataset or because I deemed the information unhelpful (e.g. the slug features contained all the categories in a specific format) : 
        ts_technologies: "jobs", "companies", "companies_found_last_week", "technology_id", "slug", "category_slug", "parent_category_slug" 
        bd_technologies: "headquarters", "seller_description", "overview", "software_product_id".
5. standardized the text in all resulting columns by lowercasing, removing special characters and removing leading and tailing spaces. 
6. outer joined the 2 datasets. 

# Product Deduplication / Entity Resolution
I first create blocks of products (and then pairs) to be checked for similarity to reduce the O(n^2) comparison space. 
I create the blocks based the first word of product names. I thought this is the most reliable piece of information about potential duplicates. However, it does result in large blocks with about 600 entities to be checked for companies with a lot of products like Adobe.

I then have a 2-step hybrid matching pipeline where I:
- first calculate the Jaro-Winkler similarity between names. If this is 1, this is a match. If it's below 0.6 I consider it a no-match (products are distinct). 
For cases above 0.6 but not 1, I then compute the cosine similarity on TD-IDF tokenized descriptions (or categories if descriptions are missing). Cosine similarities above 0.8 are deemed matches.
The thresholds were mostly chosen based on intuition and checking what other people in the field do.



## Assumptions 
- creating the blocks based on matching the first word in name. 
- the url initially seemed like a good feature to use, but it mostly links to a parent company page rather than specific product page
- product names begin with brand/model identifiers
- descriptions contain meaningful product details.


## Results
We started with 108172 entities from the merged datasets. After applying the matching pipeline we're left with 93633 products in the master catalogue, leading to 14539 products removed, or 13.4% reduction. 


## Improvements 
- find better/additional predicates for blocking. I could have also included an exact or partial categories match in addition to the first word in the name. 
- the initial name blocking could have used a text similarity check on name or other features and use entities with similarities above a certain threshold to build the blocks 
- removing stop words before computing the TF-IDF cosine similarity
- better handling of missing values. For instance inputing missing urls with the top results of a search of the product name through a web search api. 
- parallel processing for the matching pipeline.
- limited use of some of the information in the original datasets (e.g. categories, url, headquarters)
- objective evaluation of the quality of the results is currently missing. Maybe here clustering would reveal the quality of the algorithm: cluster the dataset before entity resolution and compare e.g. average cosine distance to the resulting cluster after entity resolution. An increased distance or number of clusters/size of clusters would mean more distinct entities. 
- implementing active learning or labeling some of the data would probably improve results. 


