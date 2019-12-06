# <center>CodeSimilarity v. 2</center>

*Updated December 6th, 2019*





## Synopsis:

We ran the entire CodeSimilarity v.2 pipeline on only the "winning" (or correct) submissions of CodeHunt's **Sector 2 Level 5** dataset (where students found maximum difference between elements in a given integer array). By "entire pipeline" we mean:	

1. Collecting all unique, concrete tests resulting from running Pex on all submissions

 	2. Collecting all path conditions (PCs) from re-running Pex on all submissions, this time using only the unique, concrete tests as seeded inputs
 	3. Clustering submissions by PC, according to Z3



## Preliminary Results:

Of the 44 winning submissions from the Sector 2 Level 5 dataset, our pipeline produced **five** clusters.

#### Cluster Zero

Cluster zero has 29 submissions and are each in one of the following forms:

``` c#
// Used default min(), max() functions
using System;
using System.Linq;
public class Program {
  public static int Puzzle(int[] a) {
    return a.Max()-a.Min();
  }
}
```

```c#
// Implemented min(), max() themselves
using System;
public class Program {
  public static int Puzzle(int[] a) {
	  int min = int.MaxValue;
	  int max = int.MinValue;
	  foreach(int i in a){
		  if (i>max)max=i;
		  if (i<min)min=i;
	  }
    return max-min;
  }
}
```



#### Cluster One

Cluster one has two submissions and are both in the following form:

```c#
// Two pointer method. Iterate through only half of array.
using System;
public class Program {
  public static int Puzzle(int[] a) {
    Array.Sort(a);
	int currentMax=0;
	int current=0;
	int end = a.Length-1; 
	for(int i=0;i<a.Length/2;++i)
	{
		current = a[end-i]-a[i];
		if(current > currentMax)currentMax=current;
	}
	return currentMax;
  }
}
```



#### Cluster Two

Cluster two has 10 submissions and are all in the following form:

``` c#
// Sort, then subtract.
using System;
using System.Collections;
public class Program {
  public static int Puzzle(int[] a) {
   Array.Sort(a);
    return a[a.Length-1]-a[0];
  }
}
```



#### Cluster Three

Cluster three has one submission:

```c#
// Pairwise comparison
using System;
public class Program {
  public static int Puzzle(int[] a) {
	  int max=0;
	  foreach(int i in a){
		foreach(int j in a){
		  max=i-j>max?i-j:max;
	  }
	  }
    return max;
  }
}
```



#### Cluster Four 

Cluster four has two submissions:

``` c#
// Bubble sort
using System;
public class Program {
  public static int Puzzle(int[] arr) {
    int n=arr.Length;
    int temp = 0; 
      for (int write = 0; write < arr.Length; write++) { 
        for (int sort = 0; sort < arr.Length - 1; sort++) { 
          if (arr[sort] > arr[sort + 1]) { 
            temp = arr[sort + 1]; 
            arr[sort + 1] = arr[sort]; 
            arr[sort] = temp; 
          } 
        }
      }
    return arr[n-1]-arr[0];
  }
}
```

``` c#
// Bubble sort
using System;
public class Program {
  public static int Puzzle(int[] a) {
    if (a.Length > 2) {
      for (int i = 0; i < a.Length; i++) {
        for (int j = 0; j < a.Length - 1; j++) {
		  if (a[j] > a[j + 1]) {
		    int temp = a[j];
            a[j] = a[j + 1];
            a[j + 1] = temp;
          }
        }
      }
      int max = a[a.Length-1];
      int min = a[0];
      //if (a.Length == 20) { return max;}
      return max-min;
    }
    else if (a.Length == 2)
      return a[0] > a[1] ? a[0] - a[1] : a[1] - a[0];
    return 0;
  }
}
```





