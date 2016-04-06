---
layout: post
title: Sphinx Search for dummies
category: Google Cloud Platform
tags: [Google App Enging, Sphinx Search]
---
Hi folks! [Last time](http://www.denisigo.com/2013/11/installing-sphinx-search-on-google-compute-engine-and-cloud-sql/) I showed you how to install Sphinx Search on Google Compute Engine. Today I gonna show you how Sphinx Search works. I'll use [Smuge](http://www.smuge.com) project I'm working on as a live example. 

<!--more-->

**NOTE: I’m trying to explain Fulltext search - the complex subject that is relatively new to me. I admit that my solutions may be not so good as they could be and mainly based on experiments. It needs to learn Sphinx features and other search-related things more carefully to be able to make search better. I hope to post another posts in the future to show how it goes. I also hope that you guys will help me with this.**


# What is Smuge

Smuge is a search engine for people and groups of people. People and groups have name, description and photo. People and groups may have several categories assigned to them. They also may have links to their social networks profiles (Facebook, Twitter etc), personal blog, sites, etc. Smuge has a crawler which goes through these links and collects various metrics - such as amount of followers likes, etc. 

Then, special algorithm is used to calculate so called Fame - a number showing how famous is particular person or group. That number is using only inside the system and never exposed to the users. Special A-Z Famerank letter is using instead. This letter is determined by using thresholds - for example, fame is 0 to 100 so Famerank is Z, fame is 900 to 1000 so Famerank is A and so on.

# Task

We need to perform search on persons and groups by their name, description and tags using mix of fame and search query relevance as a ranking factor. Since Smuge is search engine itself we must get the best results possible. We also need to search on categories by their name and description. So, let's start. Search is running on popular search engine (SE) named Sphinx - [http://sphinxsearch.com/](http://sphinxsearch.com/). Reference manual is placed here - [http://sphinxsearch.com/docs/current.html](http://sphinxsearch.com/docs/current.html)

# Indexes and Documents

Documents (Persons, Groups, Categories) are stored in so called Indexes as a set of named fields. These fields are basically not the same as in the main DB due to optimize speed and minimize index size. Fields can be of 2 types - fulltext and attribute:

*   Fulltext fields are used for full-text search by some pattern (basically it is the query entered by user). For example, fulltext fields can be Name, Alias, Description and so on. Sphinx is very powerful exactly in fulltext search, so fulltext fields are the main search things.
*   Attribute fields are used for just storing some useful data associated with the stored document and for filtering results retrieved with searching by fulltext fields. Filtering whole index by attribute fields is not so fast as fulltext search, so using only attribute fields to search is not recommended (for a while at least).

At the time we’re using only fulltext fields to search and a few attribute fields to store data and filter. Here is the fields set for Person, Group and Category documents (Person and Group are pretty similar so we’re using the same set). **Person/Group (let’s call them Subject as one)**

<table>

<tbody>

<tr>

<td>id</td>

<td>system</td>

<td>ID of the Subject on Smuge</td>

</tr>

<tr>

<td>name</td>

<td>fulltext</td>

<td>name of the person/group</td>

</tr>

<tr>

<td>alias</td>

<td>fulltext</td>

<td>alias (alternative name) of the person/group</td>

</tr>

<tr>

<td>description</td>

<td>fulltext</td>

<td>description of the person/group</td>

</tr>

<tr>

<td>cat_1  
cat_2  
cat_3  
cat_4  
cat_5  
cat_6</td>

<td>fulltext</td>

<td>Names of the categories Subject belongs to in corresponding order. Every category is stored in the separate field to be able to weight every category differently. Name is stored together with possible alternative names of category, so every field looks this way for example: “Guitarist Guitar player Guitar virtuoso“</td>

</tr>

<tr>

<td>slug</td>

<td>attribute</td>

<td>Name suitable for using in URLs. For example, name “Joe Satriani” is converted to slug “joe-satriani”</td>

</tr>

<tr>

<td>famerank</td>

<td>attribute</td>

<td>Famerank letter A-Z</td>

</tr>

<tr>

<td>fame</td>

<td>attribute</td>

<td>Fame number</td>

</tr>

<tr>

<td>picture_url</td>

<td>attribute</td>

<td>URL of the profile picture</td>

</tr>

<tr>

<td>data</td>

<td>attribute</td>

<td>Various data stored in the JSON format</td>

</tr>

</tbody>

</table>

**Category** Category documents have fields “id”, “name” and “slug”, which are similar to Subject’s ones. Indexes are synchronized with the main DB in some time (basically 24 h). More about indexing [here](http://sphinxsearch.com/docs/current.html#indexing).

# How search works

When search engine (SE) does a search, it performs matching of every fulltext field with query string composed using special query syntax. During this, SE calculates a lot of various factors, such as amount of keyword matches or position of the first match. These factors are used in calculation of document’s weight by the ranker. Ranker is just a formula which uses calculated factors and other available data (which also can be stored in attribute fields) to calculate document’s weight which is just a number describing how good particular document matches query. Weight is usually used to sort results. So, lets go deeper.

# Queries to SE

To perform a search, it is necessary to compose a SE query string (don’t confuse with fulltext query string described above) - string containing of what, how and where SE should search. Although Sphinx provides several ways to make requests, on Smuge we’re using one SQL-like syntax of Sphinx called SphinxQL. Composing of the request string generally is hidden in the code of Smuge, but it is important to understand how it works, so here is the real query string for searching for Person/Group.

``` sql
SELECT name, slug, picture_url, famerank, data, weight(), (weight() + fame*0.09) AS wgt
FROM `Person`
WHERE match('(@!name {query}) | (@name {query}*)')
ORDER BY wgt DESC 
LIMIT 25
OPTION ranker=expr('sum((1*lcs+(0.001/min_hit_pos)+exact_hit)*user_weight)*1000+bm25'),
field_weights=(name=5000, alias=5000, description=1000, cat_1=1000,cat_2=500,cat_3=250,cat_4=100,cat_5=50,cat_6=50);
```

This is the SELECT statement - main command telling SE to perform a search. SELECT statement consists of several clauses which we’ll describe below. Full syntax of SELECT can be found [here](http://sphinxsearch.com/docs/current.html#sphinxql-select). Note: SphinxQL also contains a lot of other statements to manage SE and indexes and do other stuff. 

**SELECT** 
SELECT is the clause describing which fields of document to return. Please notice two strange fields - weight() and (weight() + fame*0.09) with alias name “wgt” for convenience. Weight() is exactly the weight calculated by ranker. Wgt is so called “final weight” which is calculated from ranker’s weight() and fame of the Subject. I’ve not used fame right in ranker formula to divide “how good document matches query” and “how fame is affecting weight” things. 

**_Important_**: Don’t ask me why “weight() + fame*0.09” looks like this =). I’ve spent some time to adjust this and ranker formulas to make search returns relatively good results. It still needs a lot of researching and tuning. But I’ll try to explain my thoughts about how I’ve got these formulas later. 

**FROM** 
FROM clause just describes which index use to search. 

**WHERE** WHERE clause is one of the main clauses, which defines rules which determine should document be returned in results or not. Fulltext search is performed using special MATCH function. Filtering by attribute fields is performed by comparison operators. Rules can be combined using boolean operators. There should be only one MATCH function. 

Since we’re not using filtering by attribute fields, let’s describe only MATCH function. It should contain query string composed using special [fulltext query syntax](http://sphinxsearch.com/docs/current.html#extended-syntax). 

So, here is our query string:

```
(@!name {query}) | (@name {query}*)
```

Please note {query} things - they are just replaced with actual search term inside Smuge code, so if we’re searching for “music” our query string will look like this:

```
(@!name music) | (@name music*)
```

So, it contains two parts in parentheses divided by “\|” - boolean operator “OR”, which means that document will be considered as matched if whether first or second part matches. 

Note “@name” things - they define field to which perform matching. If no field defined it supposed that all fulltext fields will be matched. “@!” thing defines that all fulltext fields should be matched except defined. So, this part tells SE that it should check all fields except “name” to match “music” string exactly. 

Second part just tells SE to check only name field to match “music” string, but note “*” at the end. It means that not only exact form of “music” should match, but “musician”, “musical” etc too. 

So, this query string will match queries like “g”, “ga”, “gag”, “gaga” (search suggestions) only in name field. Other fields will match only full words. 

I’ve made such query string because if use just “{query}*” as query string, when searching for “gaga” letter by letter, instead of obvious result “Lady Gaga” there may appear some results which have some irrelevant words strarting with “g”, “ga” etc in the description but have high fame. Such behaviour is possible because we’re sorting by “wgt” field, which makes famous subjects rank higher even if they have lower weight(). 

**ORDER BY** 
ORDER BY defines sorting field and mode - ASC for ascending and DESC for descending. ORDER BY wgt DESC tells SE to sort results by wgt field (defined in the SELECT clause) in descending order. 

**LIMIT** LIMIT defines amount of results to return. 

**OPTION** 
OPTION clause is the another important one. It is used to define various settings needed for the query. Settings is written as key=value pairs separated with commas. We use two settings - **ranker** and **field_weights**: 

**ranker** defines the formula of the ranker which is used to weight results, and it’s result is stored in weight() field. It can be either name of the [builtin rankers](http://sphinxsearch.com/docs/current.html#builtin-rankers) or formula of own ranker placed inside “expr() function”. We use own formula, based on some builtin rankers. As I described above, inside ranker formula we can use a lot of factors [calculated by the SE](http://sphinxsearch.com/docs/current.html#ranking-factors). 

Factors can be document-level and field-level. Document-level factors are calculated for whole document, and field-level factors are calculated for particular fields. But, SE doesnt allow to manipulate factors for particular fields and prescribes to use some aggregate functions to get overall value for all field-level factors used. 

So, our ranker formula is:

```
sum( (1*lcs + (0.001/min_hit_pos) + exact_hit) * user_weight) * 1000 + bm25
```

here are two parts - first one is the sum() and second is bm25. 

bm25 is the statistical document-level factor showing how frequent keywords of the query string is appearing in the whole document. 

sum() is the aggregate function, which is used to sum up field-level factors collected for for all fulltext fields. So, lets describe parts of this strange forumla:

*   **1*lcs**. Lcs is the “Longest Common Subsequence between query and document, in words”, showing how good field matches query string. 1 multiplier eariler was something like 2 or 4, but became 1 during experiments.
*   **(0.001/min_hit_pos)**. min_hit_pos is “first matched occurrence position, in words, 1-based”, showing position of the first match of the query string in the field. 0.001 is to reverse its meaning and to lower its influence to overall value. “To reverse” means that its value will be bigger the nearer to the begining of the field match occurred. And, “to lower influence” means that it never will be bigger than 0.001.
*   **exact_hit**. It is equal to 1 if field contains query string exactly and 0 if not.
*   **user_weight**. User-defined weighting factor for each field - only way to affect influence of the particular field to the overall weight value. Defined it the field_weights setting in the OPTION clause (see below).

So, how it works. SE performs matching of query for every fulltext field. During this it calculates various factors for particular field and for whole document. Then it launches ranker formula. If there is aggregate function which uses field-level factors, it goes through all fields and uses formula inside that function to calculate weight value for every field using field-level factors calculated for that field and user-defined weight factor (if present) for that field. Then it aggregates these values (sum in our case), appends the rest of the ranker formula and gets overall weight value for the document. This value is presented in the weight() field. 

**field_weights** is just the array of field_name=value pairs separated with commas. Value is just the number which used in the ranker formula for that field (see ranker setting above). If value for particular field is not presented, it is by default equals to 1. 

These weight values:

```
name=5000, alias=5000, description=1000, cat_1=1000,cat_2=500,cat_3=250,cat_4=100,cat_5=50,cat_6=50
```

have this meaning: weight matches in name and alias more than anywhere else (if searched for name). Weight matches in description and first category more than in other categories.

# Explaining of the solutions

It turned out for me that the main issue is to find right balance between weight() and fame. During my tests I’ve faced several typical cases and it seems like I’ve solved them. Maybe it helps you to imagine the kind of issues and to understand solutions I’ve made: When I searched for “australian actress” there was strange document in the results “Anna Kournikova”. It turned out that she has just one word “australian” at the end of the description but has huge fame that outweighted other results with lower fame but better weight() (better matching to the query). I changed “wgt” formula to make fame not so influential and added “lcs” factor to the ranker formula to make documents which have exact “australian actress” more weighted than documents with just “australian” and even than “australian [a couple of dozens of words] actress”. 

When I searched for “actress” there was some results which have issue like previous one. They have descriptions like “Elisabeth Wong is singer and daughter of the famous actress Bobo Wong”. While Elisabeth is just singer, she appeared in the results at the top positions because of she has “actress” in the description and a lot of fame. So, I changed weights for description and cat_1 this way - if “actress” is appeared in description and in cat_1 she gets x2 of weight, but if “actress” is appeared just in one either description or cat_1, she gets only x1 of weight. 

When I searched for someone by name letter by letter (I partially described this issue above in WHERE clause) there was a lot of strange results which didn’t have letters I typed in names at all. It turned out, that they had a lot of words in description starting with what I typed. Also, they had big fame which helped them to bubble up in results. So, I changed match() function in the WHERE clause so it matches name field and others differently - name field is considered matched if contains words starting with query, but other fields must contain exact form of the query. A possible drawback of this approach is if you type name or other word letter by letter, it will be found in name field but won’t be in description until it typed fully (and correctly).
