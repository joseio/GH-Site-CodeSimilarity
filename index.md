# <center>CodeSimilarity v. 2</center>

*Updated December 2nd, 2019*





#### Synopsis:

We ran the entire CodeSimilarity v.2 pipeline on only the "winning" (or correct) submissions of CodeHunt's **Sector 2 Level 5** dataset (where students found maximum difference between elements in a given integer array). By "entire pipeline" we mean:	

	1. Collecting all unique, concrete tests resulting from running Pex on all submissions
 	2. Collecting all path conditions (PCs) from re-running Pex on all submissions, this time using only the unique, concrete tests as seeded inputs
 	3. Clustering submissions by PC, according to Z3

#### Preliminary Results:

For those concrete tests of length 20, we found (by manual inspection):

* **Four** false positives (i.e., four submissions incorrectly clustered)
* **12** *questionable* submissions (i.e., 12 submissions used O(n) strategy, but were still clustered with those that used O(2n) strategy)

For the second bullet, it's important to note that the 12 questionable submissions had the same "idea" as those submissions that they were clustered with, but they had different time complexities (i.e., approaches).





