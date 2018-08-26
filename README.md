# The overall objective
We are working on an extension that alerts users of the issue in an article. Screenshots of it working and not working are available here: https://tlkh.github.io/fake-news-chrome-extension/

### Models implemented
Timothy has trained/adopted models to detect whether the title is clickbait, the article is subjective or objective, whether the article contains toxic language, whether the article is written in the style of problematic articles like conspiracies or junk science. As they are ML models, the inference is computationally intensive, Timothy is looking for ways to make the implementation scalable. One of the methods used is to cache the results of URLs submitted.

### Limitations 
These are learnt with examples. Due to the state of the art and the small sample size, the accuracy is limited.  Moreover, the judgement on the above characteristics is also subjective as well. Therefore, these models only serve to provide advice to the readers.

Now we are focusing on disseminating the result of fact-checking articles. ClaimReview is a schema that is adopted by fact-checking websites so that it be integrated into results from search engines.

# The task here - GIVEN TEXT DETECT CLAIM
Given a list of documented and researched dubious claims (such as those from fact-checking websites like Snopes), and an article headline and/or article text, determine whether the article headline and/or article text mentions any of the listed documented and researched dubious claims. The way it is mentioned - discusses, agrees, disagrees - does not matter for now.

Our currently implemented solution: We used a simple lemmatised word match github.com/seatgeek/fuzzywuzzy to compare the headline with the entire list of article claims. This is already integrated into the extension. However, we want an improved model, because the article may not necessarily be related at all to the claim despite the existence of keywords.

The need to analyse text as well: Moreover, we want to examine the text for the documented dubious claims as well. Most the statistics and events quoted an article are likely not featured in the headline, we want to alert the readers for that as well. We are inspired by FullFact's youtu.be/ddf0lgPCoSo approach to fake-news.

A recent competition held by fakenewschallenge.org seeks for models that classify whether a piece of text agrees, disagrees, discusses or is unrelated to a piece of a headline. I have tried one of the winning solutions on a set of anti-government articles github.com/tonghuikang/fnc-ucl/blob/master/testing.ipynb. At the bottom, it features a colour-plot of the text of the headline of the article against documented local controversies. I do not have the ground truth because I have yet to find time or people to create an annotated dataset for a local dataset, and also I want to make sure such effort are spent on a meaningful task.

### Comparing every sentence in the article with every claim in the database
I feel that is it necessary to compare every sentence because it helps to identify the questionable statement made by the article, and also it makes more intuitive sense as I doubt a fixed length vector can encompass all the information from the text. 

#### Scalability
Considering the documented false claims can be as many as ten thousand, and the number of sentences or SVO triplet in an article can be a few hundred, we may need to do as many as a million computations. To make this sustainable it is likely either we use elementary methods or comparing lemmatised keywords and/or sentence vectors. 

#### Different methods tried
I feel that the comparison of sentence vectors is only meaningful when the two sentence means the same thing. "Kenya-born Obama is taking our guns away." contains the statement that "Obama is born in Kenya" (which is documented to be false) and therefore should be alerted to the reader. I suspect the cosine similarity of the vectors will be different. What was done is to use Stanford CoreNLP to extract all subject-verb-object (SVO) relationship and compute its sentence vector. However, the results were not that great. github.com/tonghuikang/similarity_corenlp-svo_sent-vec-sif/blob/master/dubious_claims.ipynb Again, we have not made the ground truth for comparison. 

The sentence vectors were computed (from the full sentence or the SVO triplet) with the averaging of word vectors. The sentence vectors could also be computed with Facebook InferSent. We can also consider computing the sentence vectors with a CNN auto-encoder that was said to work quite well in the reconstruction, or fine-tuning the ELMo layer for our downstream task for semantic similarity.

Few other recommendations to this approach: False positives could be reduced if some essential keywords are missing from each claim - for instance, we should not detect the claim of "Trump donating his salaries" if there is nothing about Trump or POTUS in the sentence. Another thing to note that is if we compare a sentence individually with the database, coreference needs to be resolved by replacing pronouns with the individuals.

# Breakdown of the process
Okay, we are given a piece of text of variable length as well as a list of claims.

## Text processing

### Processing the text as a whole
This means that we compute some document vector or compare it with the list of dubious claims. We do not feel that this is appropriate. It does not make intuitive mathematical sense that a fixed length vector can contain all the information from the article.

### Processing the text sentence by sentence
This means that we compare each sentence with the database of dubious claims. This makes more sense. I feel that is it necessary to compare every sentence because it helps to identify the questionable statement made by the article for scrutiny, as well as to provide error correction. 

### Processing the text into Subject-Verb-Object relationships
One issue with comparing each sentence to the database of claim is that the sentences are not the same. "Obama is born in Kenya" should be detected in "Kenya-born Obama is taking our guns away", however it doesn't. Therefore one way is that we could process the sentence from the text into "Obama is born in Kenya" and "Obama is taking our guns away" with CoreNLP. A demo is shown here: 

### The coreference issue
The claim that "Donald Trump mocked a disabled reporter" is covered in these two sentences "Donald Trump is an idiot. He mocked a disabled reporter.", but not only in either of these two sentences. We need to replace the "He" in the second sentence with "Donald Trump".

## What to compare for similarity

### With sentence vectors.
We could take the dot product of sentence vectors, calculated with InferSent of Priceton's SIF paper. However, once again, "Obama is born in Kenya" has a different meaning compared to "Kenya-born Obama is taking our guns away". 

### With sentence vectors of SVO triplets
This requires good extraction of the SVO triplets. CoreNLP does not extract any SVO triplets for some sentences, and that is an issue.

### Keyphrases
We have word vector embeddings but not for phrases. The phrase vector is simply calculated with max pooling of its constituent words.

## How to compare sentence/phrase similarity
Currently implemented solution lemmatised sentences with keywords. It works pretty well as per https://tlkh.github.io/fake-news-chrome-extension/. However, it is easily defeated with synonyms.

Therefore we should employ word vectors to include synonyms as well, hopefully without sacrificing false positives. Another important idea is asymmetry. When the sentence mentions the claim, it does not mean that the claim mentions the sentence.

The method I propose is to extract the keywords for both the claims and every sentence in the text. Then check if the set of keywords from any sentence in the text mainly covers all the keywords of any of the claim, if it does it is a match. The keywords are compared with the dot product of its word vectors.

Again this does not cover the intent of the sentence. For instance "Obama is born in Kenya" is detected in "Obama visit the place of birth of the President of Kenya". Order-reliant Sentence vectors may be useful to distinguish this as seen by the lower compared to peers scored. However, sentence vectors are problematic <cite notebook>

https://www.youtube.com/watch?v=ERibwqs9p38 But first we need to understand what word vectors actually mean.


# List of repositories
https://github.com/tonghuikang/given-text-detect-claim-summary <BR>
This repo summarising all the attempts that I have made.

## NLP models
https://github.com/tonghuikang/fake-news-nlp-models <BR>
A fork of Timothy's NLP models for the quality judgement of the article.

## Local databases
https://github.com/tonghuikang/str-stances <BR>
I crawled statetimesreview for their articles, and I hope to count the number of times they mentioned "millionaire ministers".

https://github.com/tonghuikang/sg-fact-check-db <BR>
This is supposed to serve as my database where all the false claims are being referenced from.

https://github.com/tonghuikang/detect-controversies-sg-keyword <BR>
I engineered keywords to some claims. The claim is detected if it contains a set of lemmatised keywords. However, this method is not scalable. I also made this into an API which worked (while the server was on).

## fakenewschallenge (FNC-1) winning solutions
https://github.com/tonghuikang/fnc-athene_system <BR>
One of the winning entries for fakenewschallenge <BR>
https://github.com/tonghuikang/fnc-athene_system/blob/master/testing.ipynb

https://github.com/tonghuikang/fnc-ucl <BR>
Another winning entry. This one is preferred because it is lightweight. <BR>
https://github.com/tonghuikang/fnc-ucl/blob/master/testing.ipynb


## Sentence similarity comparison
https://github.com/tonghuikang/sematch <BR>
https://github.com/tonghuikang/sematch/blob/master/word_sim_evaluation.ipynb <BR>
Using the WordNet to extract the claims. However it is quite useless for our objective because it will be useless for words that are not explicitly registered, and also for other reasons (https://youtu.be/ERibwqs9p38?t=7m10s) why WordNet is merely supplementary to word embeddings learnt through corpus.

https://github.com/tonghuikang/spaCy-experiments <BR>
Just trying out code from spaCy. Nothing much.

https://github.com/tonghuikang/InferSent <BR>
Trying out InferSent sentence embeddings.  <BR>
https://github.com/tonghuikang/InferSent/blob/master/encoder/demo.ipynb

https://github.com/tonghuikang/sent-vec_skip-thoughts <BR>
Another neural network method to generate sentence vectors.


## Extraction of SVO triplets
https://github.com/tonghuikang/similarity_corenlp-svo_sent-vec-spacy <BR>
Using spaCy to find SVO triplets. Not very effective.

https://github.com/tonghuikang/similarity_corenlp-svo_sent-vec-sif <BR>
https://github.com/tonghuikang/similarity_corenlp-svo_sent-vec-sif/blob/master/dubious_claims.ipynb <BR>
The code embeddings SIF embeddings are found in this repo:
https://github.com/tonghuikang/sent_vec-SIF

https://github.com/tonghuikang/svo-nltk <BR>
An attempt to use the code found online to extract SVO.

https://github.com/tonghuikang/Stanford-OpenIE-Python <BR>
This is a Python wrapper for Stanford OpenIE to extract the SVO triples. We could directly fire up the CoreNLP server and call for it instead.


## Current solution
???
