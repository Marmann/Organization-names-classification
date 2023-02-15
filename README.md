# How to run the code
The code is in the form of a Jupyer notebook which is a adapted here in an exploratory an prototyping phase, and also because it provided in this case a useful graphic tool to participate in the refining of analysis even without getting deep inside the code.

Indispensable prerequisites : 
Having ‘Case_study_names_mapping.csv’ in your working directory (the CSV version)
Download the GLEIF Golden Data and Copy files as CSV on this page : https://www.gleif.org/en/lei-data/gleif-golden-copy/download-the-golden-copy#/  
Level 1 LEI-CDF Golden Copy File
Level 2 Relationship Record (RR) CDF Golden Copy File
In the notebook, change the filenames to match with the copies you will have downloaded, as the database is updated every eight hours

Optional but preferable to save yourself some of the work : 
Have the pre-prepared word files ‘indiscriminative_words.csv’ and ‘irrelevant_words.csv’ in your working directory


Then everything should be pretty linear.

# Performance analysis of the algorithms


## Processing of GLEIF Data : 

This is done in O(n^2) at max with n being the maximum of the sizes of the subdatasets ‘lei2’ and ‘rr’

The merge is the part that has the biggest computational complexity, as it is in O(n * m), where n is the number of rows in the first input file and m is the number of rows in the second input file. This is because the function reads both input files row by row and writes a constant number of columns (3) to the output file for each row of the second input file

Removing duplicates should be optimized in Python so it should be done in linear or superlinear time, but it cannot go over O(n2)

All of this part should only be done once and does not depend on the size of our data (the list of organizations)


## Trie (prefix tree) structure and processing of GLEIF data with tries :

The time complexity a of building a trie from a set of keys is O(n * m), where n is the number of keys and m is the maximum key length. This is because each key must be inserted into the trie, and each insertion takes time proportional to the length of the key.
As a result, since our keys have a limited length, we can consider that it is done in linear time. 

The basic methods of the Trie class all have an O(h) complexity with h the height of the tree, however the height is equal to the max length of keys, and since our data has bound lengths of strings for the company names, we can also consider it is constant.

The find_nearest_with_parent() method has a time complexity in the worst case of O(m * N), where m is the length of the company name being searched and N is the number of nodes in the trie (N is is equal to k^h at max, with k being the length of the alphabet and h the height of the tree). This is because the method performs a depth-first search of the trie, visiting each node once and taking time proportional to the length of the company name to search for a given node.
In the worst case, the method will visit all nodes in the trie and return the path to the last node, so the time complexity is O(N * m). However, in the best case, the method will find a node with a parent company in the first few steps and return the path to that node, so the time complexity will be closer to O(m).
In general, we expect the time complexity to be around O(m) since the tree is densely populated. In practice this works because the distance between closest neighbours is bound by ~200 in our tests.

Updating the tree with children has a complexity of O(N) since we perform a depth first search browsing the complete tree.
Updating the tree for parent company name with the help of nearest neighbors will have an O(N * N * m) worst case complexity but this is not realistic, we are rather around a O(N * m) time complexity because the tree is densely populated. In practice, that is the case for the reasons mentioned earlier.


## Parent company matching
We then work with dataframes, we perform searches with the help of tries.
Building on what we said before, the complexity of virtually all operations on dataframes will have an O(n*m) complexity, with n being the size of our data, because we iterate over the n rows of the dataframes, and the operations on words are done in O(m) with m being the length of the company name.

The clustering part using the scikit_learn library is done in a max of O(n^2) time but following my calculations, it should rather be in O(n^3/2) because we perform the clustering on small different chunks of data that add up to the total data, and we do not perform clustering on the whole data.

## A word on spatial complexity
Dealing with datasets of this size do not pose a threat to computer memory, even up until millions of companies, as we have seen with the GLEIF dataset.
However, this area could be investigated further upon if we try to reduce space complexity.

## To go further
We could do a further analysis on the current dataset to evaluate the time that the algorithms effectively take and evaluate how it would scale on bigger data.

## Conclusion
All in all, we have a complexity which should be bound by O( max(n^3/2, n*m,k^m*m)) 
with n the size ouf our dataset, m the max length of the strings, k the size of the alphabet.

This means that our approach is scalable to a larger dataset, 100 000 data points should not be an issue providing we don’t expect the result in the minute, but within a few minutes it should be okay.




# Our approach

## Global review
What we did here was to get a large dataset with companies and parent companies, the GLEIF dataset. After cleaning this dataset to extract what we need, we try to find matches between organizations in our data and in the GLEIF dataset, to pair our organizations with potential parent companies.

We then filter these matches to keep only the most relevant, with a combination of criteria including but not restricted to : 
number of common words between the organization and the potential parent organization
length of longest common subsequence, in absolute terms and in comparison with the length of the organization name 
the presence of very usual words in organization companies, such as legal suffixes, country names, connection words… More on that later
We even calculate a closeness score to help the future user to better identify the problematic cases.

Based on that, we move on to round 2 of our quest for connections between companies, and we use an affinity propagation clustering approach, with the use of a Levenshtein distance for strings, to find groups of organizations that are supposedly linked. Then we compute the longest subsequence of each group to find a potential group name, and we finally filter these findings once again to provide with a definitive mapping of the companies.

# A few words on the interactive part and the frequent words analysis
In the Jupyter notebook, you’ll notice there is an interactive section to let the user decide which frequent words to filter in the analysis.
I have segmented them in two sets : the first one, stored in the ‘to_remove’ list, is composed of words that are deemed to be no relevant whatsoever in the comparison between company names. These are words such as ‘inc’, ‘gmbh’, and others (not restricted to legal suffixes).
The second set comprises words that can be relevant for the matching but not on their own. For instance take country names (France, UK), types of organizations (Association, Agency), types of business (Consulting, Car dealing) . A lot of organizations have these types of words in their name, however between two organizations who only have one such word in common, we shouldn't say that they are linked. For instance ‘Boston Consulting Group’ and ‘ABeam Consulting’ are not linked. However, with two or more such words in common, it can very well be the case, and eliminating these words completely from our analysis would do us a disservice. Take ‘Agence France Presse’, the three words seem very frequent, however they must absolutely be taken into account for pairing companies.
I have put some sets of rules to help us decide to eliminate or to keep a potential connection, notably the number of common words and the difference between the length of the longest common subsequence and the max length of common words.

## What could be further done
There is quite a few fine tuning that can be done on all of these criteria that i put in place, because they are subjective and do not stem from a long rigorous and detailed analysis of all of the different parameters to take into account.

More precisely, and most significantly, I have hopes for the closeness score and I think we can find a way to learn the best parameters for this function, for it to be able to discriminate effectively between pairs that have a link and pairs who do not. I thought of doing a logistic regression to find the parameters of this function, with the help of the GLEIF dataset, from which we can directly extract data of linked companies, and from which we can construct new data of pairs of companies who we know for sure do not have a link.
With a bit more time, and after finetuning the details of the current approach, I would explore this avenue first.

Also and perhaps more importantly, we could work on further pre-building sets of frequent words, perhaps by category (geographical locations and nationalities, type of organization, line of business, first names…), to refine our analysis.
