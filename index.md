# <center>CodeSimilarity v. 2</center>

*Updated December 3rd, 2019*





## Synopsis:

We ran the entire CodeSimilarity v.2 pipeline on only the "winning" (or correct) submissions of CodeHunt's **Sector 2 Level 5** dataset (where students found maximum difference between elements in a given integer array). By "entire pipeline" we mean:	

1. Collecting all unique, concrete tests resulting from running Pex on all submissions

 	2. Collecting all path conditions (PCs) from re-running Pex on all submissions, this time using only the unique, concrete tests as seeded inputs
 	3. Clustering submissions by PC, according to Z3



## Preliminary Results:

For those concrete tests of length 20, we found (by manual inspection):

* **Three** false positives (i.e., three submissions incorrectly clustered)
* **Two** *questionable* submissions (i.e., two submissions grouped together, even though one used the *O(n)* strategy, whereas the other used the *O(2n)* strategy)

For the second bullet, it's important to note that the two questionable submissions were grouped into **cluster 4** but should have been grouped into **cluster 0** (because they employ the same strategy). Again, one of the two submissions employed *O(n)* strategy while the other used the *O(2n)*.



### False Positives

<span style='color:red'>**TODO:**</span> How do we define these clusters, by time complexity (i.e., num. loops), or by logical equivalence. This is crucial, as this directly impacts false positive rate.

### Questionable submissions

The questionable submissions employing the *O(2n)* strategy is:

```c#
using System;
public class Program {
  public static int Puzzle(int[] a) {
	 int min=a[0],max=a[0];
	 int i=0,k=0;
		 while((i+1)!=a.Length) {
		  if(a[i+1]>max) max = a[i+1];
		  i++;
		 }
		  while((k+1)!=a.Length) {
		  if(a[k+1]<min) min = a[k+1];
		  //else min=a[k];
		  k++;
		 }
    return max-min;
  }
}
```



The questionable submission employing the *O(n)* strategy is:

```c#
using System;
public class Program {
  public static int Puzzle(int[] a) {
	  int max=a[0],min=a[0];
	  foreach(int i in a) {
		  max=i>max?i:max;
		  min=i<min?i:min;
	  }
    return max-min;
  }
}
```



These two questionable submissions should've been placed into **cluster 0**, which follows the following pattern:

```c#
using System;
using System.Linq;
public class Program {
  public static int Puzzle(int[] a) {
	return a.Max()-a.Min();
  }
}
```







