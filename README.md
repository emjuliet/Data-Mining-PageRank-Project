# Pagerank Project

In this project, you will create a simple search engine for the website <https://www.lawfareblog.com>.
This website provides legal analysis on US national security issues.

**Due date:** Sunday, 18 September at midnight

**Computation:**
This project has low computational requirements.
You are not required to complete it on the lambda server (although you are welcome to if you'd like).

## Background

**Data:**

The `data` folder contains two files that store example "web graphs".
The file `small.csv.gz` contains the example graph from the *Deeper Inside Pagerank* paper.
This is a small graph, so we can manually inspect the contents of this file with the following command:
```
$ zcat data/small.csv.gz
source,target
1,2
1,3
3,1
3,2
3,5
4,5
4,6
5,6
5,4
6,4
```

> **Recall:**
> The `cat` terminal command outputs the contents of a file to stdout, and the `zcat` command first decompressed a gzipped file and then outputs the decompressed contents.

As you can see, the graph is stored as a CSV file.
The first line is a header,
and each subsequent line stores a single edge in the graph.
The first column contains the source node of the edge and the second column the target node.
The file is assumed to be sorted alphabetically.

The second data file `lawfareblog.csv.gz` contains the link structure for the lawfare blog.
Let's take a look at the first 10 of these lines:
```
$ zcat data/lawfareblog.csv.gz | head
source,target
www.lawfareblog.com/,www.lawfareblog.com/topic/interrogation
www.lawfareblog.com/,www.lawfareblog.com/upcoming-events
www.lawfareblog.com/,www.lawfareblog.com/
www.lawfareblog.com/,www.lawfareblog.com/our-comments-policy
www.lawfareblog.com/,www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
www.lawfareblog.com/,www.lawfareblog.com/topic/lawfare-research-paper-series
www.lawfareblog.com/,www.lawfareblog.com/topic/book-reviews
www.lawfareblog.com/,www.lawfareblog.com/documents-related-mueller-investigation
www.lawfareblog.com/,www.lawfareblog.com/topic/international-law-loac
```
You can see that in this file, the node names are URLs.
Semantically, each line corresponds to an HTML `<a>` tag that is contained in the source webpage and links to the target webpage.

We can use the following command to count the total number of links in the file:
```
$ zcat data/lawfareblog.csv.gz | wc -l
1610789
```
Since every link corresponds to a non-zero entry in the `P` matrix,
this is also the value of `nnz(P)`.
(Technically, we should subtract 1 from this value since the `wc -l` command also counts the header line, not just the data lines.)

To get the dimensions of `P`, we need to count the total number of nodes in the graph.
The following command achieves this by: decompressing the file, extracting the first column, removing all duplicate lines, then counting the results.
```
$ zcat data/lawfareblog.csv.gz | cut -f1 -d, | uniq | wc -l
25761
```
This matrix is large enough that computing matrix products for dense matrices takes several minutes on a single CPU.
Fortunately, however, the matrix is very sparse.
The following python code computes the fraction of entries in the matrix with non-zero values:
```
>>> 1610788 / (25760**2)
0.0024274297384360172
```
Thus, by using sparse matrix operations, we will be able to speed up the code significantly.

**Code:**

The `pagerank.py` file contains code for loading the graph CSV files and searching through their nodes for key phrases.
For example, you can perform a search for all nodes (i.e. urls) that mention the string `corona` with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --search_query=corona
```

> **NOTE:**
> It will take about 10 seconds to load and parse the data files.
> All the other computation happens essentially instantly.

Currently, the pagerank of the nodes is not currently being calculated correctly, and so the webpages are returned in an arbitrary order.
Your task in this assignment will be to fix these calculations in order to have the most important results (i.e. highest pagerank results) returned first.

## Task 1: the power method

Implement the `WebGraph.power_method` function in `pagerank.py` for computing the pagerank vector by fixing the `FIXME` annotation.

**Part 1:**

To check that your implementation is working,
you should run the program on the `data/small.csv.gz` graph.
For my implementation, I get the following output.
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=2.5629e-01
DEBUG:root:i=1 residual=1.1841e-01
DEBUG:root:i=2 residual=7.0701e-02
DEBUG:root:i=3 residual=3.1815e-02
DEBUG:root:i=4 residual=2.0497e-02
DEBUG:root:i=5 residual=1.0108e-02
DEBUG:root:i=6 residual=6.3716e-03
DEBUG:root:i=7 residual=3.4228e-03
DEBUG:root:i=8 residual=2.0879e-03
DEBUG:root:i=9 residual=1.1750e-03
DEBUG:root:i=10 residual=7.0131e-04
DEBUG:root:i=11 residual=4.0321e-04
DEBUG:root:i=12 residual=2.3800e-04
DEBUG:root:i=13 residual=1.3812e-04
DEBUG:root:i=14 residual=8.1083e-05
DEBUG:root:i=15 residual=4.7251e-05
DEBUG:root:i=16 residual=2.7704e-05
DEBUG:root:i=17 residual=1.6164e-05
DEBUG:root:i=18 residual=9.4778e-06
DEBUG:root:i=19 residual=5.5066e-06
DEBUG:root:i=20 residual=3.2042e-06
DEBUG:root:i=21 residual=1.8612e-06
DEBUG:root:i=22 residual=1.1283e-06
DEBUG:root:i=23 residual=6.1907e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
```
Yours likely won't be identical (due to weird floating point issues), but it should be similar.
In particular, the ranking of the nodes/urls should be the same order.

> **NOTE:**
> The `--verbose` flag causes all of the lines beginning with `DEBUG` to be printed.
> By default, only lines beginning with `INFO` are printed.

**Part 2:**

The `pagerank.py` file has an option `--search_query`, which takes a string as a parameter.
If this argument is used, then the program returns all nodes that match the query string sorted according to their pagerank.
Essentially, this gives us the most important pages related to our query.

Again, you may not get the exact same results as me,
but you should get similar results to the examples I've shown below.
Verify that you do in fact get similar results.

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
```

Most of these pages are not very interesting, however, because they are not articles,
and usually when we are performing a web search, we only want articles.

This raises the question: How can we find the most important articles filtering out the non-article pages?
The answer is to modify the `P` matrix by removing all links to non-article pages.

This raises another question: How do we know if a link is a non-article page?
Unfortunately, this is a hard question to answer with 100% accuracy,
but there are many methods that get us most of the way there.
One easy to implement method is to compute what's called the "in-link ratio" of each node (i.e. the total number of edges with the node as a target divided by the total number of nodes),
and then remove nodes from the search results with too-high of a ratio.
The intuition is that non-article pages often appear in the menu of a webpage, and so have links from almost all of the other webpages;
but article-webpages are unlikely to appear on a menu and so will only have a small number of links from other webpages.
The `--filter_ratio` parameter causes the code to remove all pages that have an in-link ratio larger than the provided value.

Using this option, we can estimate the most important articles on the domain with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
```
Notice that the urls in this list look much more like articles than the urls in the previous list.

When Google calculates their `P` matrix for the web,
they use a similar (but much more complicated) process to modify the `P` matrix in order to reduce spam results.
The exact formula they use is a jealously guarded secret that they update continuously.

In the case above, notice that we have accidentally removed the blog's most popular article (<www.lawfareblog.com/snowden-revelations>).
The blog editors believed that Snowden's revelations about NSA spying are so important that they directly put a link to the article on the menu.
So every single webpage in the domain links to the Snowden article,
and our "anti-spam" `--filter-ratio` argument removed this article from the list.
In general, it is a challenging open problem to remove spam from pagerank results,
and all current solutions rely on careful human tuning and still have lots of false positives and false negatives.

**Part 4:**

Recall from the reading that the runtime of pagerank depends heavily on the eigengap of the `\bar\bar P` matrix,
and that this eigengap is bounded by the alpha parameter.

Run the following four commands:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
```
You should notice that the last command takes considerably more iterations to compute the pagerank vector.
(My code takes 685 iterations for this call, and about 10 iterations for all the others.)

This raises the question: Why does the second command (with the `--alpha` option but without the `--filter_ratio`) option not take a long time to run?
The answer is that the `P` graph for <https://www.lawfareblog.com> naturally has a large eigengap and so is fast to compute for all alpha values,
but the modified graph does not have a large eigengap and so requires a small alpha for fast convergence.

Changing the value of alpha also gives us very different pagerank rankings.
For example, 
```
$ python3 pagerank_solution.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

$ python3 pagerank_solution.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
INFO:root:rank=0 pagerank=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```

Which of these rankings is better is entirely subjective,
and the only way to know if you have the "best" alpha for your application is to try several variations and see what is best.
If large alphas are good for your application, you can see that there is a trade-off between quality answers and algorithmic runtime.
We'll be exploring this trade-off more formally in class over the rest of the semester.

## Task 2: the personalization vector

The most interesting applications of pagerank involve the personalization vector.
Implement the `WebGraph.make_personalization_vector` function so that it outputs a personalization vector tuned for the input query.
The pseudocode for the function is:
```
for each index in the personalization vector:
    get the url for the index (see the _url_to_index function)
    check if the url satisfies the input query (see the url_satisfies_query function)
    if so, set the corresponding index to one
normalize the vector
```

**Part 1:**

The command line argument `--personalization_vector_query` will use the function you created above to augment your search with a custom personalization vector.
If you've implemented the function correctly,
you should get results similar to:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```

Notice that these results are significantly different than when using the `--search_query` option:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --search_query='corona'
INFO:root:rank=0 pagerank=8.1320e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=1 pagerank=7.7908e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=2 pagerank=5.2262e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=3 pagerank=3.9584e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=3.8114e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=3.3973e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=6 pagerank=3.3633e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus
INFO:root:rank=7 pagerank=3.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 pagerank=3.2160e-03 url=www.lawfareblog.com/congress-needs-coronavirus-failsafe-its-too-late
INFO:root:rank=9 pagerank=3.1036e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
```

Which results are better?
Again, that depends on what you mean by "better."
With the `--personalization_vector_query` option,
a webpage is important only if other coronavirus webpages also think it's important;
with the `--search_query` option,
a webpage is important if any other webpage thinks it's important.
You'll notice that in the later example, many of the webpages are about Congressional proceedings related to the coronavirus.
From a strictly coronavirus perspective, these are not very important webpages.
But in the broader context of national security, these are very important webpages.

Google engineers spend TONs of time fine-tuning their pagerank personalization vectors to remove spam webpages.
Exactly how they do this is another one of their secrets that they don't publicly talk about.

**Part 2:**

Another use of the `--personalization_vector_query` option is that we can find out what webpages are related to the coronavirus but don't directly mention the coronavirus.
This can be used to map out what types of topics are similar to the coronavirus.

For example, the following query ranks all webpages by their `corona` importance,
but removes webpages mentioning `corona` from the results.
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
```
You can see that there are many urls about concepts that are obviously related like "covid", "clinical trials", and "quarantine",
but this algorithm also finds articles about Chinese propaganda and Trump's policy decisions.
Both of these articles are highly relevant to coronavirus discussions,
but a simple keyword search for corona or related terms would not find these articles.

<!--
**Part 3:**

Select another topic related to national security.
You should experiment with a national security topic other than the coronavirus.
For example, find out what articles are important to the `iran` topic but do not contain the word `iran`.
Your goal should be to discover what topics that www.lawfareblog.com considers to be related to the national security topic you choose.
-->

## Submission

1. Create a new repo on github (not a fork of this repo).

1. Run the following commands, and paste their output into the code blocks below.
   
   Task 1, part 1:
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=0.2869156002998352
DEBUG:root:i=1 residual=0.17673587799072266
DEBUG:root:i=2 residual=0.11748470366001129
DEBUG:root:i=3 residual=0.061109527945518494
DEBUG:root:i=4 residual=0.038023941218853
DEBUG:root:i=5 residual=0.018694842234253883
DEBUG:root:i=6 residual=0.011767178773880005
DEBUG:root:i=7 residual=0.009790820069611073
DEBUG:root:i=8 residual=0.01014517992734909
DEBUG:root:i=9 residual=0.010316635482013226
DEBUG:root:i=10 residual=0.00993178877979517
DEBUG:root:i=11 residual=0.009173526428639889
DEBUG:root:i=12 residual=0.008200429379940033
DEBUG:root:i=13 residual=0.007159791421145201
DEBUG:root:i=14 residual=0.006137470249086618
DEBUG:root:i=15 residual=0.005187023431062698
DEBUG:root:i=16 residual=0.004333975724875927
DEBUG:root:i=17 residual=0.0035879577044397593
DEBUG:root:i=18 residual=0.0029477779753506184
DEBUG:root:i=19 residual=0.0024063601158559322
DEBUG:root:i=20 residual=0.001953764818608761
DEBUG:root:i=21 residual=0.0015790157485753298
DEBUG:root:i=22 residual=0.0012710248120129108
DEBUG:root:i=23 residual=0.0010196113726124167
DEBUG:root:i=24 residual=0.0008153740200214088
DEBUG:root:i=25 residual=0.000650388712529093
DEBUG:root:i=26 residual=0.0005175543483346701
DEBUG:root:i=27 residual=0.0004110064182896167
DEBUG:root:i=28 residual=0.00032570521580055356
DEBUG:root:i=29 residual=0.0002577237319201231
DEBUG:root:i=30 residual=0.00020367132674437016
DEBUG:root:i=31 residual=0.00016063386283349246
DEBUG:root:i=32 residual=0.0001265958562726155
DEBUG:root:i=33 residual=9.965635399566963e-05
DEBUG:root:i=34 residual=7.833473500795662e-05
DEBUG:root:i=35 residual=6.153235153760761e-05
DEBUG:root:i=36 residual=4.832382910535671e-05
DEBUG:root:i=37 residual=3.799118348979391e-05
DEBUG:root:i=38 residual=2.9711076422245242e-05
DEBUG:root:i=39 residual=2.3324044377659447e-05
DEBUG:root:i=40 residual=1.8230430214316584e-05
DEBUG:root:i=41 residual=1.4268070117395837e-05
DEBUG:root:i=42 residual=1.1168278433615342e-05
DEBUG:root:i=43 residual=8.670030183566269e-06
DEBUG:root:i=44 residual=6.788734026486054e-06
DEBUG:root:i=45 residual=5.372772648115642e-06
DEBUG:root:i=46 residual=4.164867732470157e-06
DEBUG:root:i=47 residual=3.2266434573102742e-06
DEBUG:root:i=48 residual=2.5501456093479646e-06
DEBUG:root:i=49 residual=1.9844856069539674e-06
DEBUG:root:i=50 residual=1.537638127047103e-06
DEBUG:root:i=51 residual=1.2043236665704171e-06
DEBUG:root:i=52 residual=8.70921553541848e-07
INFO:root:rank=0 pagerank=8.5429e-01 url=4
INFO:root:rank=1 pagerank=6.7512e-01 url=6
INFO:root:rank=2 pagerank=5.4108e-01 url=5
INFO:root:rank=3 pagerank=3.1644e-01 url=2
INFO:root:rank=4 pagerank=2.5502e-01 url=3
INFO:root:rank=5 pagerank=2.3246e-01 url=1
```

   Task 1, part 2:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=3.9236e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=3.4359e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=2.3217e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=2.2610e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=2.1226e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=2.0721e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=2.0332e-03 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=1.9729e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=1.9693e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=9 pagerank=1.8279e-03 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='Trump'
INFO:root:rank=0 pagerank=4.2801e-02 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=3.4683e-02 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=2 pagerank=2.7849e-02 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=2.6351e-02 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=4 pagerank=2.4786e-02 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=5 pagerank=2.2742e-02 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=2.0582e-02 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=1.8467e-02 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=1.7373e-02 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=1.7181e-02 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors
   
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='Iran'
INFO:root:rank=0 pagerank=3.4590e-02 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=2.3338e-02 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=1.4032e-02 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.1128e-02 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=6.9530e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=5 pagerank=6.9320e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=6 pagerank=6.8211e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=6.0697e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=5.8303e-03 url=www.lawfareblog.com/trump-moves-cut-irans-oil-revenues-whats-his-endgame
INFO:root:rank=9 pagerank=4.9612e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
``` 
  
  Task 1, part 3:
``` 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=3.6346e+00 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=1 pagerank=3.6346e+00 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=3.6346e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban 
INFO:root:rank=3 pagerank=3.6346e+00 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=4 pagerank=3.6346e+00 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=5 pagerank=3.6346e+00 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=6 pagerank=3.6346e+00 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=3.6346e+00 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=8 pagerank=3.6346e+00 url=www.lawfareblog.com/topics
INFO:root:rank=9 pagerank=3.6346e+00 url=www.lawfareblog.com/documents-related-mueller-investigation

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=1.5650e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=1.1968e+00 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=1.1837e+00 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=5.6602e-01 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony        
INFO:root:rank=4 pagerank=5.6297e-01 url=www.lawfareblog.com/senate-examines-threats-homeland
INFO:root:rank=5 pagerank=4.8742e-01 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
INFO:root:rank=6 pagerank=4.8691e-01 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
INFO:root:rank=7 pagerank=4.8294e-01 url=www.lawfareblog.com/whats-house-resolution-impeachment
INFO:root:rank=8 pagerank=4.4849e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=9 pagerank=4.4788e-01 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
``` 
  
  Task 1, part 4:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=17.440872192382812
DEBUG:root:i=1 residual=2.352112054824829
DEBUG:root:i=2 residual=1.032855749130249
DEBUG:root:i=3 residual=1.4071199893951416
DEBUG:root:i=4 residual=1.1935572624206543
DEBUG:root:i=5 residual=0.9074719548225403
DEBUG:root:i=6 residual=0.6669377684593201
DEBUG:root:i=7 residual=0.48432767391204834
DEBUG:root:i=8 residual=0.3507046699523926
DEBUG:root:i=9 residual=0.2535971403121948
DEBUG:root:i=10 residual=0.18326784670352936
DEBUG:root:i=11 residual=0.13241557776927948
DEBUG:root:i=12 residual=0.09568120539188385
DEBUG:root:i=13 residual=0.06912985444068909
DEBUG:root:i=14 residual=0.04994850233197212
DEBUG:root:i=15 residual=0.03608337789773941
DEBUG:root:i=16 residual=0.026073016226291656
DEBUG:root:i=17 residual=0.01883767545223236
DEBUG:root:i=18 residual=0.0136115038767457
DEBUG:root:i=19 residual=0.00983438640832901
DEBUG:root:i=20 residual=0.007100687362253666
DEBUG:root:i=21 residual=0.0051336404867470264
DEBUG:root:i=22 residual=0.003707722993567586
DEBUG:root:i=23 residual=0.002679175930097699
DEBUG:root:i=24 residual=0.0019356489647179842
DEBUG:root:i=25 residual=0.0014003089163452387
DEBUG:root:i=26 residual=0.001011196873150766
DEBUG:root:i=27 residual=0.0007286579930223525
DEBUG:root:i=28 residual=0.0005287293461151421
DEBUG:root:i=29 residual=0.0003800259146373719
DEBUG:root:i=30 residual=0.0002759307681117207
DEBUG:root:i=31 residual=0.00019909966795239598
DEBUG:root:i=32 residual=0.0001420972985215485
DEBUG:root:i=33 residual=0.00010491906868992373
DEBUG:root:i=34 residual=7.352768443524837e-05
DEBUG:root:i=35 residual=5.6175689678639174e-05
DEBUG:root:i=36 residual=4.0481296309735626e-05
DEBUG:root:i=37 residual=2.974073686345946e-05
DEBUG:root:i=38 residual=1.9828041331493296e-05
DEBUG:root:i=39 residual=1.3219462744018529e-05
DEBUG:root:i=40 residual=9.087840226129629e-06
DEBUG:root:i=41 residual=8.260465619969182e-06
DEBUG:root:i=42 residual=7.434663075400749e-06
DEBUG:root:i=43 residual=3.3058386179618537e-06
DEBUG:root:i=44 residual=1.6524658121852553e-06
DEBUG:root:i=45 residual=4.129655280848965e-06
DEBUG:root:i=46 residual=2.4784383185760817e-06
DEBUG:root:i=47 residual=8.292935262943502e-07
INFO:root:rank=0 pagerank=3.6346e+00 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=1 pagerank=3.6346e+00 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=3.6346e+00 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban 
INFO:root:rank=3 pagerank=3.6346e+00 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=4 pagerank=3.6346e+00 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=5 pagerank=3.6346e+00 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=6 pagerank=3.6346e+00 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=3.6346e+00 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=8 pagerank=3.6346e+00 url=www.lawfareblog.com/topics
INFO:root:rank=9 pagerank=3.6346e+00 url=www.lawfareblog.com/documents-related-mueller-investigation
   
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=20.522029876708984
DEBUG:root:i=1 residual=3.2541022300720215
DEBUG:root:i=2 residual=1.682108759880066
DEBUG:root:i=3 residual=2.6968038082122803
DEBUG:root:i=4 residual=2.6924362182617188
DEBUG:root:i=5 residual=2.4082977771759033
DEBUG:root:i=6 residual=2.0822181701660156
DEBUG:root:i=7 residual=1.7801107168197632
DEBUG:root:i=8 residual=1.5160776376724243
DEBUG:root:i=9 residual=1.2895281314849854
DEBUG:root:i=10 residual=1.0963331460952759
DEBUG:root:i=11 residual=0.931947648525238
DEBUG:root:i=12 residual=0.7921093106269836
DEBUG:root:i=13 residual=0.6732890605926514
DEBUG:root:i=14 residual=0.5722895860671997
DEBUG:root:i=15 residual=0.48643964529037476
DEBUG:root:i=16 residual=0.4134327173233032
DEBUG:root:i=17 residual=0.351414293050766
DEBUG:root:i=18 residual=0.29867827892303467
DEBUG:root:i=19 residual=0.2538731098175049
DEBUG:root:i=20 residual=0.21581155061721802
DEBUG:root:i=21 residual=0.18343177437782288
DEBUG:root:i=22 residual=0.155913308262825
DEBUG:root:i=23 residual=0.13249918818473816
DEBUG:root:i=24 residual=0.11257273703813553
DEBUG:root:i=25 residual=0.09568405151367188
DEBUG:root:i=26 residual=0.08129733055830002
DEBUG:root:i=27 residual=0.06907977908849716
DEBUG:root:i=28 residual=0.05873076617717743
DEBUG:root:i=29 residual=0.04992436245083809
DEBUG:root:i=30 residual=0.042432427406311035
DEBUG:root:i=31 residual=0.03608115762472153
DEBUG:root:i=32 residual=0.030668431892991066
DEBUG:root:i=33 residual=0.02606748230755329
DEBUG:root:i=34 residual=0.022158194333314896
DEBUG:root:i=35 residual=0.018834175541996956
DEBUG:root:i=36 residual=0.01600644364953041
DEBUG:root:i=37 residual=0.013605224899947643
DEBUG:root:i=38 residual=0.011564300395548344
DEBUG:root:i=39 residual=0.009830544702708721
DEBUG:root:i=40 residual=0.008355551399290562
DEBUG:root:i=41 residual=0.007101740222424269
DEBUG:root:i=42 residual=0.006036021281033754
DEBUG:root:i=43 residual=0.005130247212946415
DEBUG:root:i=44 residual=0.004360362887382507
DEBUG:root:i=45 residual=0.003706395858898759
DEBUG:root:i=46 residual=0.0031504365615546703
DEBUG:root:i=47 residual=0.0026778499595820904
DEBUG:root:i=48 residual=0.0022762645967304707
DEBUG:root:i=49 residual=0.0019349028589203954
DEBUG:root:i=50 residual=0.0016444288194179535
DEBUG:root:i=51 residual=0.0013977537164464593
DEBUG:root:i=52 residual=0.0011880823876708746
DEBUG:root:i=53 residual=0.0010095367906615138
DEBUG:root:i=54 residual=0.0008580518187955022
DEBUG:root:i=55 residual=0.0007290771463885903
DEBUG:root:i=56 residual=0.0006196887115947902
DEBUG:root:i=57 residual=0.0005267090746201575
DEBUG:root:i=58 residual=0.0004476915637496859
DEBUG:root:i=59 residual=0.00038053488242439926
DEBUG:root:i=60 residual=0.0003233944298699498
DEBUG:root:i=61 residual=0.0002748763363342732
DEBUG:root:i=62 residual=0.00023366123787127435
DEBUG:root:i=63 residual=0.00019860932661686093
DEBUG:root:i=64 residual=0.0001688184420345351
DEBUG:root:i=65 residual=0.00014354585437104106
DEBUG:root:i=66 residual=0.00012201457866467535
DEBUG:root:i=67 residual=0.00010371280222898349
DEBUG:root:i=68 residual=8.817000343697146e-05
DEBUG:root:i=69 residual=7.49422688386403e-05
DEBUG:root:i=70 residual=6.3699008023832e-05
DEBUG:root:i=71 residual=5.414372208178975e-05
DEBUG:root:i=72 residual=4.602066837833263e-05
DEBUG:root:i=73 residual=3.9116664993343875e-05
DEBUG:root:i=74 residual=3.324855788378045e-05
DEBUG:root:i=75 residual=2.8260448743822053e-05
DEBUG:root:i=76 residual=2.4021228455239907e-05
DEBUG:root:i=77 residual=2.0424808099051006e-05
DEBUG:root:i=78 residual=1.7418316929251887e-05
DEBUG:root:i=79 residual=1.4839864888926968e-05
DEBUG:root:i=80 residual=1.2625874660443515e-05
DEBUG:root:i=81 residual=1.0731567272159737e-05
DEBUG:root:i=82 residual=9.122446499532089e-06
DEBUG:root:i=83 residual=7.754150828986894e-06
DEBUG:root:i=84 residual=6.590978046006057e-06
DEBUG:root:i=85 residual=5.602069450105773e-06
DEBUG:root:i=86 residual=4.7618091230106074e-06
DEBUG:root:i=87 residual=4.047406946483534e-06
DEBUG:root:i=88 residual=3.440305818003253e-06
DEBUG:root:i=89 residual=2.9243708468129626e-06
DEBUG:root:i=90 residual=2.4856847176124575e-06
DEBUG:root:i=91 residual=2.1125499642948853e-06
DEBUG:root:i=92 residual=1.7958890339286881e-06
DEBUG:root:i=93 residual=1.5264253079294576e-06
DEBUG:root:i=94 residual=1.297501626140729e-06
DEBUG:root:i=95 residual=1.1028663493561908e-06
DEBUG:root:i=96 residual=9.376781804348866e-07
INFO:root:rank=0 pagerank=5.6271e-04 url=www.lawfareblog.com/masthead
INFO:root:rank=1 pagerank=5.6271e-04 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=5.6271e-04 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban 
INFO:root:rank=3 pagerank=5.6271e-04 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=4 pagerank=5.6271e-04 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=5 pagerank=5.6271e-04 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=6 pagerank=5.6271e-04 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=7 pagerank=5.6271e-04 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=8 pagerank=5.6271e-04 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=9 pagerank=5.6271e-04 url=www.lawfareblog.com/topics
   
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=4.349486351013184
DEBUG:root:i=1 residual=2.1895055770874023
DEBUG:root:i=2 residual=1.4083446264266968
DEBUG:root:i=3 residual=0.7250767946243286
DEBUG:root:i=4 residual=0.4009532928466797
DEBUG:root:i=5 residual=0.3034133315086365
DEBUG:root:i=6 residual=0.2522180676460266
DEBUG:root:i=7 residual=0.20211829245090485
DEBUG:root:i=8 residual=0.155814990401268
DEBUG:root:i=9 residual=0.11721795052289963
DEBUG:root:i=10 residual=0.08690471947193146
DEBUG:root:i=11 residual=0.06383287161588669
DEBUG:root:i=12 residual=0.0465703085064888
DEBUG:root:i=13 residual=0.033789075911045074
DEBUG:root:i=14 residual=0.024398690089583397
DEBUG:root:i=15 residual=0.017543086782097816
DEBUG:root:i=16 residual=0.012567486613988876
DEBUG:root:i=17 residual=0.008975998498499393
DEBUG:root:i=18 residual=0.006397310644388199
DEBUG:root:i=19 residual=0.004553572274744511
DEBUG:root:i=20 residual=0.0032404980156570673
DEBUG:root:i=21 residual=0.0023089756723493338
DEBUG:root:i=22 residual=0.0016487521352246404
DEBUG:root:i=23 residual=0.0011817582417279482
DEBUG:root:i=24 residual=0.0008507650345563889
DEBUG:root:i=25 residual=0.0006154624279588461
DEBUG:root:i=26 residual=0.00044793725828640163
DEBUG:root:i=27 residual=0.0003278690273873508
DEBUG:root:i=28 residual=0.00024153481354005635
DEBUG:root:i=29 residual=0.000178758185938932
DEBUG:root:i=30 residual=0.00013277480320539325
DEBUG:root:i=31 residual=9.88419633358717e-05
DEBUG:root:i=32 residual=7.38308735890314e-05
DEBUG:root:i=33 residual=5.527903340407647e-05
DEBUG:root:i=34 residual=4.133297625230625e-05
DEBUG:root:i=35 residual=3.089193705818616e-05
DEBUG:root:i=36 residual=2.3114491341402754e-05
DEBUG:root:i=37 residual=1.7379241398884915e-05
DEBUG:root:i=38 residual=1.298186725762207e-05
DEBUG:root:i=39 residual=9.706667697173543e-06
DEBUG:root:i=40 residual=7.219230610644445e-06
DEBUG:root:i=41 residual=5.393935680331197e-06
DEBUG:root:i=42 residual=3.999141881649848e-06
DEBUG:root:i=43 residual=3.014574303961126e-06
DEBUG:root:i=44 residual=2.2342508145811735e-06
DEBUG:root:i=45 residual=1.6447568214061903e-06
DEBUG:root:i=46 residual=1.208479375236493e-06
DEBUG:root:i=47 residual=9.109922984862351e-07
INFO:root:rank=0 pagerank=1.5650e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=1.1968e+00 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=1.1837e+00 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=5.6602e-01 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony        
INFO:root:rank=4 pagerank=5.6297e-01 url=www.lawfareblog.com/senate-examines-threats-homeland
INFO:root:rank=5 pagerank=4.8742e-01 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
INFO:root:rank=6 pagerank=4.8691e-01 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
INFO:root:rank=7 pagerank=4.8294e-01 url=www.lawfareblog.com/whats-house-resolution-impeachment
INFO:root:rank=8 pagerank=4.4849e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=9 pagerank=4.4788e-01 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
   
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=5.116973400115967
DEBUG:root:i=1 residual=3.0303795337677
DEBUG:root:i=2 residual=2.2931981086730957
DEBUG:root:i=3 residual=1.388965129852295
DEBUG:root:i=4 residual=0.9036080837249756
DEBUG:root:i=5 residual=0.8044591546058655
DEBUG:root:i=6 residual=0.7867180705070496
DEBUG:root:i=7 residual=0.7416884899139404
DEBUG:root:i=8 residual=0.6726692318916321
DEBUG:root:i=9 residual=0.5953382253646851
DEBUG:root:i=10 residual=0.5192638635635376
DEBUG:root:i=11 residual=0.44870713353157043
DEBUG:root:i=12 residual=0.38512733578681946
DEBUG:root:i=13 residual=0.328741192817688
DEBUG:root:i=14 residual=0.2792661190032959
DEBUG:root:i=15 residual=0.2362288236618042
DEBUG:root:i=16 residual=0.19908957183361053
DEBUG:root:i=17 residual=0.16728615760803223
DEBUG:root:i=18 residual=0.1402565985918045
DEBUG:root:i=19 residual=0.11745154857635498
DEBUG:root:i=20 residual=0.0983428880572319
DEBUG:root:i=21 residual=0.08243181556463242
DEBUG:root:i=22 residual=0.06925473362207413
DEBUG:root:i=23 residual=0.05838729441165924
DEBUG:root:i=24 residual=0.04944784194231033
DEBUG:root:i=25 residual=0.04209873825311661
DEBUG:root:i=26 residual=0.03604709729552269
DEBUG:root:i=27 residual=0.03104379028081894
DEBUG:root:i=28 residual=0.026881448924541473
DEBUG:root:i=29 residual=0.02339104190468788
DEBUG:root:i=30 residual=0.02043755166232586
DEBUG:root:i=31 residual=0.01791498064994812
DEBUG:root:i=32 residual=0.015741214156150818
DEBUG:root:i=33 residual=0.013853137381374836
DEBUG:root:i=34 residual=0.012202413752675056
DEBUG:root:i=35 residual=0.01075174193829298
DEBUG:root:i=36 residual=0.009472141042351723
DEBUG:root:i=37 residual=0.008340616710484028
DEBUG:root:i=38 residual=0.007338651455938816
DEBUG:root:i=39 residual=0.006450895685702562
DEBUG:root:i=40 residual=0.0056644282303750515
DEBUG:root:i=41 residual=0.0049681090749800205
DEBUG:root:i=42 residual=0.004352181684225798
DEBUG:root:i=43 residual=0.003808018984273076
DEBUG:root:i=44 residual=0.0033279096242040396
DEBUG:root:i=45 residual=0.002904914552345872
DEBUG:root:i=46 residual=0.0025327936746180058
DEBUG:root:i=47 residual=0.002205914119258523
DEBUG:root:i=48 residual=0.0019191931933164597
DEBUG:root:i=49 residual=0.0016680596163496375
DEBUG:root:i=50 residual=0.0014483965933322906
DEBUG:root:i=51 residual=0.0012565169017761946
DEBUG:root:i=52 residual=0.0010891151614487171
DEBUG:root:i=53 residual=0.0009432441438548267
DEBUG:root:i=54 residual=0.0008162783342413604
DEBUG:root:i=55 residual=0.0007058868068270385
DEBUG:root:i=56 residual=0.0006100028986111283
DEBUG:root:i=57 residual=0.0005267999367788434
DEBUG:root:i=58 residual=0.00045466554001905024
DEBUG:root:i=59 residual=0.00039218016900122166
DEBUG:root:i=60 residual=0.00033809617161750793
DEBUG:root:i=61 residual=0.00029131950577721
DEBUG:root:i=62 residual=0.00025089114205911756
DEBUG:root:i=63 residual=0.00021597281738650054
DEBUG:root:i=64 residual=0.0001858324685599655
DEBUG:root:i=65 residual=0.00015983184857759625
DEBUG:root:i=66 residual=0.00013741463772021234
DEBUG:root:i=67 residual=0.00011809736315626651
DEBUG:root:i=68 residual=0.00010145954001927748
DEBUG:root:i=69 residual=8.713623537914827e-05
DEBUG:root:i=70 residual=7.48109960113652e-05
DEBUG:root:i=71 residual=6.420951831387356e-05
DEBUG:root:i=72 residual=5.509430047823116e-05
DEBUG:root:i=73 residual=4.72600877401419e-05
DEBUG:root:i=74 residual=4.052908479934558e-05
DEBUG:root:i=75 residual=3.474807090242393e-05
DEBUG:root:i=76 residual=2.9784425350953825e-05
DEBUG:root:i=77 residual=2.5523995645926334e-05
DEBUG:root:i=78 residual=2.1868181647732854e-05
DEBUG:root:i=79 residual=1.873202018032316e-05
DEBUG:root:i=80 residual=1.604245335329324e-05
DEBUG:root:i=81 residual=1.3736363143834751e-05
DEBUG:root:i=82 residual=1.1759617336792871e-05
DEBUG:root:i=83 residual=1.006554339255672e-05
DEBUG:root:i=84 residual=8.614055332145654e-06
DEBUG:root:i=85 residual=7.370667390205199e-06
DEBUG:root:i=86 residual=6.305753686319804e-06
DEBUG:root:i=87 residual=5.39390157427988e-06
DEBUG:root:i=88 residual=4.61322269984521e-06
DEBUG:root:i=89 residual=3.9449960240744986e-06
DEBUG:root:i=90 residual=3.3730857467162423e-06
DEBUG:root:i=91 residual=2.883718707380467e-06
DEBUG:root:i=92 residual=2.4650421437399928e-06
DEBUG:root:i=93 residual=2.106901320075849e-06
DEBUG:root:i=94 residual=1.8005706579060643e-06
DEBUG:root:i=95 residual=1.5386098084491096e-06
DEBUG:root:i=96 residual=1.314618543801771e-06
DEBUG:root:i=97 residual=1.1231117014176561e-06
DEBUG:root:i=98 residual=9.594132279744372e-07
INFO:root:rank=0 pagerank=3.1166e-04 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.0192e-04 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.0058e-04 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.3644e-04 url=www.lawfareblog.com/senate-examines-threats-homeland
INFO:root:rank=4 pagerank=1.2694e-04 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
INFO:root:rank=5 pagerank=1.2689e-04 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
INFO:root:rank=6 pagerank=1.2643e-04 url=www.lawfareblog.com/whats-house-resolution-impeachment
INFO:root:rank=7 pagerank=1.1941e-04 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
INFO:root:rank=8 pagerank=1.1363e-04 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony        
INFO:root:rank=9 pagerank=6.6637e-05 url=www.lawfareblog.com/events

```

   Task 2, part 1:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' 
INFO:root:rank=0 pagerank=2.4868e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=2.4866e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.1435e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global   
INFO:root:rank=3 pagerank=8.0579e-02 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=4 pagerank=8.0579e-02 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=5 pagerank=7.4815e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=6 pagerank=7.4815e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=6.6662e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections   
INFO:root:rank=8 pagerank=6.4483e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=5.6433e-02 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns

```
   Task 2, part 2:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=2.4868e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=2.4866e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.1435e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global   
INFO:root:rank=3 pagerank=6.6662e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections   
INFO:root:rank=4 pagerank=5.2454e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=4.4881e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=6 pagerank=4.2900e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=7 pagerank=3.6023e-02 url=www.lawfareblog.com/trump-cant-play-politics-aid-states
INFO:root:rank=8 pagerank=3.4500e-02 url=www.lawfareblog.com/senators-urge-cyber-leaders-prevent-attacks-healthcare-sector
INFO:root:rank=9 pagerank=3.4413e-02 url=www.lawfareblog.com/how-do-you-spy-when-world-shut-down
```

1. Ensure that all your changes to the `pagerank.py` and `README.md` files are committed to your repo and pushed to github.

1. Get at least 5 stars on your repo.
   (You made trade stars with other students in the class.)

   > **NOTE:**
   > 
   > Recruiters use github profiles to determine who to hire,
   > and pagerank is used to rank user profiles and projects.
   > Links in this graph correspond to who has starred/followed who's repo.
   > By getting more stars on your repo, you'll be increasing your github pagerank, which increases the likelihood that recruiters will hire you.
   > To see an example, [perform a search for `data mining`](https://github.com/search?q=data+mining).
   > Notice that the results are returned "approximately" ranked by the number of stars,
   > but because "some stars count more than others" the results are not exactly ranked by the number of stars.
   > (I asked you not to fork this repo because forks are ranked lower than non-forks.)
   >
   > In some sense, we are doing a "dual problem" to data mining by getting these stars.
   > Recruiters are using data mining to find out who the best people to recruit are,
   > and we are hacking their data mining algorithms by making those algorithms select you instead of someone else.
   >
   > If you're interested in exploring this idea further, here's a python tutorial for extracting GitHub's social graph: <https://www.oreilly.com/library/view/mining-the-social/9781449368180/ch07.html> ; if you're interested in learning more about how recruiters use github profiles, read this Hacker News post: <https://news.ycombinator.com/item?id=19413348>.

1. Submit the url of your repo to sakai.

   Each part is worth 2 points, for 12 points overall.
