# Topic-Modeling-Arxiv-Dataset
This is a new approach for topic modeling. Topics have been extracted from over 600,000 articles sourced from the Arxiv Dataset

Please download the notebook (analysis.ipynb) in this repository to follow along.

## Methodology

Before diving into the extraction of topics in this massive dataset, let’s explore the methodology that was used to accomplish this goal. There is no obvious or straightforward method to approach this problem. In fact, topic modeling is a very large field of study within the natural language processing community. To exemplify this, let’s say that a simple TF-IDF algorithm is applied to the entire corpus (i.e., all the abstracts). This might seem like a good place to start, after all, TF-IDF algorithms are very good at capturing the importance that words have relative to their own containing bodies of text and all other bodies of text contained in the same dataset. However, in this case, it will certainly perform poorly. This is because the algorithm will favor those words/topics that are less frequent across a massive number of unrelated bodies of text (i.e., abstracts). Consequently, the algorithm will likely fail to model the topics. This phenomenon occurs because irrelevant topics contained in small numbers of abstracts are over-represented due to their rareness relative to the large number of unrelated abstracts that don’t contain them but rather contain other more common and relevant topics. To further exemplify this, one can refer to figure 1 where the articles’ categories have been illustrated. If a topic is extracted from an economics article (a low occurring category) and compared to the rest of the corpus, it will certainly return a high TF-IDF; however, if the opposite is done and a highly occurring category is selected such as computer science, the topics extracted from the articles in this category will return very low TF-IDF scores since the inverse document frequency is dramatically reduced by the fact that there will certainly be more articles interrelated and thus with similar topics. Therefore, the algorithm could erroneously discard relevant topics present in a large number of related articles. As a result, there is a high risk of producing highly irrelevant topics that can mislead and misrepresent the true underlying structure of the dataset’s topics. To mitigate these risks and produce a fair representation of this dataset’s topics, the methodology that has been developed and implemented consists of clustering related articles before extracting any information from them. This method enables the isolated analysis of the most relevant topics within each individual cluster or group of related abstracts using a bagging technique. Then, these internal topics’ relative importance to other clusters’ topics can be assessed to filter the most significant topics produced by each cluster by using a TF-IDF approach. There are some libraries for topic modeling that have similar approaches; however, in this case, a specific and unique implementation of the described methodology has been developed from scratch by integrating some well-known clustering, dimensionality reduction, and custom-coded algorithms. 

Loading the Data
	The ArXiv dataset a is available in the JSON format and although the actual articles have been excluded, its size is over three (3) gigabytes. Due to the sheer amount of data present in this dataset, the loading and manipulation processes of the data are to be carefully considered, especially when using local computational resources (i.e., using a personal computer). Consequently, the data have been loaded using the Dask Framework, specifically, the Dask Bag class. Dask has optimized this class for parallel computing and proper management of multicore-capable CPUs and GPUs. Therefore, it yields much better results compared to simply loading the file using the standard JSON library. After having loaded the data successfully, the data will be filtered and processed before undergoing exploratory data analysis (EDA).
	
![Screenshot 2023-03-22 093436](https://user-images.githubusercontent.com/83890387/226978893-1be30db5-6180-468a-a6ff-2f7c7283c248.jpg)

1.0.1 CODE SNIPPET
```
# Dask was developed to natively scale packages like Numpy, pandas and scikit-learn and the surrounding ecosystem to multi-core machines and distributed clusters when datasets exceed memory.
# Cited from https://www.dask.org/
# Load data using Dask's Bag object. Which, according to my local environment, is about 3 times faster compared to loading the dataset strictly using the "json" library. 

loaded_data = db.read_text('arxiv-db.json').map(json.loads)
```
## Feature Extraction / Transformations

Before making use of the data to extract trending topics from it, it is necessary to improve the structure of the dataset to have a better representation of the underlying topics. The first step towards improving the data structure is to interpret the codified “category” attribute included in the dataset. This attribute describes each article’s main topics but has been codified into acronyms that provide little information and are not suitable for vectorization (i.e., transforming text into numerical representations). Therefore, the attribute has been interpreted and transformed into three additional features. Those features are the following: main category, subcategory, and category description. All these newly created features are in a human-readable format (i.e., text). To generate and populate the proposed features, a function was coded to extract, transform and load these fields. This function matches the acronym contained in the original “category” attribute with additional data published in the Arxiv dataset website found at https://arxiv.org/category_taxonomy. Performing this transformation will generate a quick insight into the distribution of the main categories included in the articles. Furthermore, once all this data has been embedded and then reduced in dimensionality, the data points that share all or some main categories, subcategories, and descriptions should be pushed closer together into more homogeneous clusters. The reasons for these considerations will be further explored in the upcoming sections. Finally, generating these features will greatly benefit the exploratory data analysis process. 

1.0.2 CODE SNIPPET
```
#These tuples will be used to match each category's acronym and generate newly proposed features.
print(*cleaned_new_data[:5], sep='\n')

('cs.ar', 'computer science', 'hardware architecture', 'Covers systems organization and hardware architecture. Roughly includes material in ACM Subject Classes C.0, C.1, and C.5.')
```

## Filtering and Cleaning Data

This dataset contains 2,207,558 data points. To yield better results, the data set will be filtered down to fewer and more relevant data points.  Relevance is an arbitrary concept in this case mainly because of the definition of a “trend”. For this case study, trends have been defined as those topics that have remained relevant and investigated for at least the past four (4) years. This filtering process will be applied to the Dask Bag Object to speed up the process. Moreover, the actual transformation of the feature discussed in the previous section was run at this stage. The result was then converted into a Panda’s data frame. Finally, the 4-year time frame considers any scientific article that has been created or updated within this period. The keyword here is “updated”. Some of the research papers in this dataset were created over a decade ago; however, if these papers keep getting updates, it is most likely because the topics of research contained in the articles are still relevant and under investigation. Therefore, the attributes considered in the filtering process were the “version” and “update_date” columns. Finally, after running the code, the data set was reduced to 662,346 data points. Moreover, the constructed data frame was populated with the newly created features. Finally, some data types were modified to match their nature (e.g., dates were transformed into date-times data types) and thus, facilitated data manipulation processes.

1.0.3 CODE SNIPPET
```
# Filter all articles that have ramained relevant or mostly investigated for at least the past 4 years
columns = ['id','category','abstract']
loaded_data = (loaded_data.filter(lambda x: get_last(x=x) > 2018).map(trim_attributes).compute())

# convert to pandas
df = pd.DataFrame(loaded_data)

# Generate new features by traforming the old category attribute and filter data
df['new_category'] = df['category'].apply(lambda x: category_extraction(x=x))
df['sub_category'] = df['category'].apply(lambda x: subcategory_extraction(x=x))
df['description'] = df['category'].apply(lambda x: description_extraction(x=x))
df['update_date'] = pd.to_datetime(df['update_date'])
df['created_date'] = pd.to_datetime(df['versions'].apply(lambda x: last_creation_date(x=x)))
df = df.reindex(columns=['id', 'title', 'authors', 'category', 'new_category', 'sub_category', 'created_date', 'update_date', 'description', 'abstract', 'versions'])
```

## Exploratory Data Analysis

Moving on, to have a better understanding of the data, a basic exploratory data analysis process has been performed on the dataset in this section. After running some code, the key takeaways from this process are the following:
1.- The “description” attribute contains some null values. This is not of concern since the only purpose of this field is to be merged into the “abstract” field in the vectorization process to add and enrich the context of each article’s abstract and thus derive better clusters or groups of related articles. 
2.- The articles’ categories have been analyzed to understand how they are distributed across the dataset. For example, it has been concluded that the category with the largest number of papers is “computer science” followed by “mathematics”. Although this report is not after understanding these categories or broader topics, These categories provide a powerful insight regarding the structure of the trending topics. Finally, the association among these categories has been studied and visualized utilizing a heatmap. This is not so relevant for clustering, dimensionality reduction, or natural language processing; however, it is certainly important for real-world applications such as implementing supervised learning algorithms to optimize the searching process across the database. For instance, the results reveal that computer science topics are highly associated with mathematics and statistics articles, which might suggest some expected high activity in fields such as data science and artificial intelligence. Although generating these supervised models is outside the scope of this article, it is worth mentioning that this data could be further expanded by analyzing the sub-categories of each article as well. 

Figure 1 - Distribution of general topics across the ArXiv dataset. 

![barplot](https://user-images.githubusercontent.com/83890387/226979137-d7f924ec-ed2d-4b80-9c59-c596609d9ceb.png)
 
Figure 2 - Visual representation of the topics’ associations across the ArXiv dataset. Each number within a cell represents the number or articles containing the associated topics.  

![heatmap](https://user-images.githubusercontent.com/83890387/226979204-37efc0c3-615c-4b45-928b-24ac4f8f95d9.png)

## Feature Engineering and Clustering

To begin modeling these clusters, the articles must be transformed into a numeric representation. To do so, the sentence-transformers library has been used. Sentence Transformers is a Python framework for state-of-the-art sentence, text, and image embeddings (SentenceTransformers Documentation, 2022). This embedding method is especially strong at creating and finding semantic similarities between sentences and documents. Although such a large dataset would have been a perfect candidate for training a model purely based on the data points available, the computational cost of training a model locally (i.e., on a personal computer) is too high. Thus, two pre-trained models have been tested as candidates to generate the embedding for these articles. These models are 'allenai-specter' and "all-MiniLM-L6-v2". The latter was trained with scientific articles and the former with general bodies of text found in sources like Wikipedia or Reddit. Before generating any embeddings, the abstracts were enhanced by adding a sentence with the following structure: “This article’s main category(ies) is(are) {main category(ies) to which the article belongs}. Some topics related to these articles include: {sub-categories}”. This sentence was automatically generated based on the features (i.e., the categories and subcategories) extracted in the previous sections. Following this sentence, the descriptions of the categories corresponding to each article were also added to the abstract in a syntactically coherent manner. The purpose of such enhancement is to aid the embedding process in pushing related vectors (i.e., the numerical representation of abstracts) together in high-dimensional vector spaces. After carefully evaluating both pre-trained models, the one that produced the best results (i.e., created the best clusters) was 'allenai-specter'. However, this model generates embedding in 768 dimensions. Before moving on, it’s important to mention that generating these embeddings takes an enormous amount of time on a local computer; therefore, it limits the number of pre-trained models and variants of the dataset that can be tested. 

1.0.4 CODE SNIPPET
```
# Data enhancement and feature engineering 
articles_text = ['[CLS]' + article[0] + '[SEP]' + article[1] for article in zip(df['title'].to_list(), df['text'].to_list())]
articles_text[:10]
model = SentenceTransformer('allenai-specter')
corpus_embeddings = model.encode(articles_text, convert_to_tensor=True, show_progress_bar=True)
```

Subsequently, before one can implement any clustering method, it is important to reduce the dimensionality of these embeddings. Otherwise, the clustering methods will perform poorly due to the so-called curse of dimensionality. In this case, UMAP or Uniform Manifold Approximation and Projection Method has been chosen as the dimensionality reduction algorithm of preference because of its capacity to work with non-linear data. Finally, after tuning this algorithm and reducing the embeddings from 768 dimensions to 15 dimensions, the clusters can now be extracted. To do so, two clustering methods were tested. The first one tested was HDBSCAN or Hierarchical Density Bases Spatial Clustering of Application with Noise, which is a hybrid between hierarchical and density-based clustering algorithms. This algorithm was chosen as a candidate due to its capacity of handling noisy data and finding clusters with amorphous shapes. Moreover, it is optimized to work with large datasets. Next, sci-kit-learn’s K-means algorithm was assessed. This is a simpler algorithm only capable of producing perfectly spherical clusters. The algorithms were evaluated using the density-based clustering validating metric for the HDBSCAN and the elbow and silhouette method for the K-means. Of course, they were also both further tested down the road by observing the quality of the topics both clustering methods generated.

1.0.5 CODE SNIPPET
```
# Dimensionality reduction and clustering
corpus_embeddings_np = corpus_embeddings.cpu().numpy().astype(np.float64)
umap_embeddings = umap.UMAP(n_neighbors=15, n_components=5, min_dist=0.0, metric='cosine').fit_transform(corpus_embeddings_np)
clusterer = hdbscan.HDBSCAN(min_cluster_size=200, metric='euclidean', cluster_selection_method='eom', prediction_data=True)
clusterer.fit(umap_embeddings)
```

The results produced from the exhausting testing phase were the following: HDSCAN created 219 clusters after finding the optimal hyperparameters. This might seem high at first but if one considers the size of the dataset, 219 clusters are totally doable. However, two aspects of this clustering method were of great concern. First, almost one-fourth of all data points were considered outliers or noise. Second, the algorithm created a cluster that included almost one-sixth of all data points. On the other hand, k-means was generating visually coherent clusters in two dimensions, but those clusters had a poor silhouette score of 0.47 at the suggested k number (i.e., the number of clusters) by the elbow method (10 clusters). To plot the clusters and visually evaluate both clustering algorithms, two-dimensional embeddings were created using the UMAP method again. However, the data points were color-coded using the clusters generated from the embeddings in 15 dimensions. After thoroughly evaluating both methods, HDBSCAN rendered better results. Nonetheless, some decisions had to be taken regarding the concerns associated to this clustering method. First, all outliers (i.e., abstracts) were removed and not considered candidates for topic modeling. This makes sense because if these documents are true outliers, the process of extracting topics from them will produce isolated topics that are, by no means, a trend. After all, the main purpose of this report is to extract trending or highly relevant topics and, since these isolated articles are not representative of any given cluster, one can safely assume that the topics they contain cannot be considered trends but rather rare occurrences. Moreover, the computational cost is of great concern when working with large datasets, thus, removing these outliers will greatly benefit the processing times. Next, after exploring the unusually sized cluster previously mentioned, it was clear that the model had clustered together these articles due to their actual similarities. As one can notice from figure 1, a considerable number of categories are related to physics but are separated into subcategories. Therefore, the clustering algorithm pushed them together into one single cluster. To prevent this cluster from causing an under-representation of topics within other much smaller clusters, this unusually large cluster was independently evaluated in the topic extraction process. Although the bagging and TF-IDF scores used to model the topics in the next section were normalized to account for the obvious fact that clusters have different sizes, clusters with very large differences in size can still cause misinterpretation of the scores. It is important to mention that a large number of data points within this large cluster do not necessarily means that the topics contained here are more relevant. It could possibly be that the original dataset is heavily used by physicists, astronomers, or other related scientists and thus contains more documents related to these topics. Also, it is possible that these fields of science publish articles much more frequently than other fields. Nonetheless, the size of the clusters could easily be account for as a weight that influences the importance of the extracted topics if one would like to do so. For the sake of this report, the topics of this unusual cluster will be reported separately. The most important topics extracted from this cluster were filtered by evaluating their relative importance to those contained in all other clusters; however, the importance of topics contained in all other clusters was not assessed in relation to this cluster to prevent misrepresentation problems. In the next section, the methods used to extract the topics will be introduced and all this will be further clarified.

Figure 3 - Visual comparison of the clustering produced by HDBSCAN and K-means. The colors represent clusters produced by embedding in 15 dimensions. 

![cluster_umap_hdbscan_labels](https://user-images.githubusercontent.com/83890387/226979336-0544b75c-33e5-4184-a63c-83272b5245bf.png)

![cluster_umap_kmeans_labels](https://user-images.githubusercontent.com/83890387/226979388-efda81c4-3ab6-42e5-854b-b68076580464.png)

## Topic Modeling

In this final section, the trending topics are extracted. To do so, the clusters will first be analyzed in isolation from each other. As a result, the most relevant topics within each cluster should emerge from this process. To begin, it is necessary to pre-process all the abstracts. Although we had created an enhanced version of the abstract, the raw or original abstract will be used at this stage since the enhancements’ sole purpose was to enrich the context within each original abstract in order to produce better clusters. The pre-processing includes the removal of none alpha-numeric characters, punctuation, unnecessary spaces, and stop words (i.e., those words that carry little semantic meaning such as articles). Moreover, all bodies of text were split into lists of words or tokens. At this point, all words were lemmatized. Lemmatizing is the process of converting words into their root word while preserving their original meaning. Lemmatization was chosen over stemming (i.e., a more aggressive method for reducing words to their roots) because of its ability to preserve the original semantic and morphological meaning although at a higher computational cost. Lemmatized words will prevent words with the same meaning from generating several topics and thus weakening their true importance. Once all abstracts have been processed, the Count-Vectorizer class available in the Sci-Kit learn library was used to extract the occurring frequency of each word within each cluster. The array returned by the fit_tranformed method of this class was then normalized using the l1 norm to account for the differences in the size of the different clusters. Finally, this data was transformed into a data frame where each row is a cluster, and each column is a word. The values in this data frame correspond to the normalized counts of occurrences of each word within each cluster. To calculate these values, all the abstracts belonging to a given cluster were joined together and individually processed. The result is a sparse matrix. 

Table 1 – Sparse matrix with normalized counts of word occurrences.

![Picture3](https://user-images.githubusercontent.com/83890387/226979474-b7c939f8-a84f-4ab2-973c-3729a7b31bad.jpg)

Next, the importance of each word within each cluster was assessed relative to all other clusters. To do so, the TF-IDF score for each word was calculated. The clusters were considered as if they were a single body of text. Therefore, the data frame constructed contains clusters as rows, words as columns, and normalized values that reflect the importance of each word contained in a given cluster relative to all other clusters. 

Table 2 – Sparse matrix with normalized TD-IDF scores

![Picture2](https://user-images.githubusercontent.com/83890387/226979516-36c1ef94-05fb-44ae-b593-b3c704ab7c1c.jpg)

Finally, to assess the overall importance of a topic, the importance of a word within a cluster (i.e., the normalized counts returned by the Count-Vectorizer class) was transformed by applying a weight to it. In this case, the weight corresponds to the TF-IDF score of that same word within the cluster and relative to all other clusters. For this dataset, the best results for this transformation came from simply multiplying these two values. However, it is worth mentioning that other transformations could be explored. 

## Conclusion.

There is certainly room for improvement by testing different hyperparameters, data structures, and transformations. Moreover, if one would like to provide more context to the topics, using n-gram vectorizing algorithms could help understand the context of the topics. N-grams algorithms consider n number of words before and after each given word and return the frequency of occurrences of those given phrases. Nonetheless, this method has yielded a very good understanding of the governing topics. The following table illustrates the topics extracted. Some topics have been color-coded to illustrate a possible relationship. 

Table 3 – Extracted Topics

![Screenshot 2023-03-22 093950](https://user-images.githubusercontent.com/83890387/226978677-a3aa8197-9348-43e4-9698-e2664d80b5f0.jpg)

To conclude, the topics generated seem to be very relevant in modern times. For example, topics such as artificial intelligence, tensors, code, and blockchain are surely areas of great public interest. Some words were manually removed from the final top 100 list because they are generic words (e.g., dialogue, opinion, game, group), that without the proper context, little inference can be extracted from them. Topic modeling is a complex task with many possible implementations. Nonetheless, the topics extracted here are certainly very relevant and insightful 

