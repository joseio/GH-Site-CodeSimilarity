<title>CodeSimilarity v. 2</title>
# <center>CodeSimilarity v. 2</center>

*Last updated January 20th, 2020*





# Contents

[Synopsis](#synopsis)

[Potential Applications of Tool](#potential-applications-of-tool)

[Use Caxses](#use-cases)

[CodeHunt Dataset](#codehunt-dataset)

[Preliminary Results](#preliminary-results)

[Sample of Results](#quick-peak-of-sector2-level5s-results)

[Cyclomatic Complexity Number (CCN)](#cyclomatic-complexity-number-ccn)

[Questions and Doubts](#questions-and-doubts)

[Algorithm Sources](#algorithm-sources)

[Bugs](#bugs)





## Synopsis:

We ran the entire CodeSimilarity v.2 pipeline on only the "winning" (or correct) submissions of CodeHunt's **Sector 2 Level 5** dataset (where students found maximum difference between elements in a given integer array). By "entire pipeline" we mean:	

1. Collecting all unique, concrete tests resulting from running Pex on all submissions

 	2. Collecting all path conditions (PCs) and symbolic return values (SRVs) from re-running Pex on all submissions, this time using only the unique, concrete tests as seeded inputs
 	3. Clustering submissions by PCs and SRVs, according to Z3





## Potential Applications of Tool:

Let's say that we have a problem where the instructor asks students to write a method, `Puzzle` in C# to find the greatest element in an integer array by implementing the **quick sort** algorithm. The method takes in an integer array. Let's say the student submissions to the problem look like this:



#### Submission one (completely correct)

Let's say that one student submits a correct quick sort implementation:

```C#
static public int Partition(int[] arr, int left, int right) {
  int pivot;
  pivot = arr[left];
  while (true) {
      while (arr[left] < pivot) 
          left++;
      while (arr[right] > pivot) 
          right--;
      if (left < right) {
          int temp = arr[right];
          arr[right] = arr[left];
          arr[left] = temp;
      } else 
          return right;
  }
}
static public void quickSort(int[] arr, int left, int right) {
  int pivot;
  if (left < right) {
      pivot = Partition(arr, left, right);
      if (pivot > 1) 
          quickSort(arr, left, pivot - 1);
      if (pivot + 1 < right) 
          quickSort(arr, pivot + 1, right);
  }
}
static int Puzzle(int[] arr) {
  quickSort(arr, 0, a.Length - 1);
  return arr[arr.Length-1];
}
```



#### Submission two (right answer, wrong algorithm)

Now let's say that another student submitted this bubble sort implementation:

```c# 
static void bubbleSort(int[] arr) {
  for (int j = 0; j <= arr.Length - 2; j++) 
    for (int i = 0; i <= arr.Length - 2; i++)
      if (arr[i] > arr[i + 1]) {
        temp= arr[i + 1];
        arr[i + 1] = arr[i];
        arr[i] = temp;
      }
}
static int Puzzle(int[] arr) {
  bubbleSort(arr);
  return arr[arr.Length - 1];
}
```



#### Submission three (right answer, wrong algorithm)

Now let's say that another student submitted this insertion sort implementation:

````c#
static void insertionSort(int[] arr) {
  for (i = 1; i < n; i++) {
    val = arr[i];
    flag = 0;
    j = i - 1;
    while(j >= 0 && flag != 1) {
      if (val < arr[j]) {
        arr[j + 1] = arr[j];
        j--;
        arr[j + 1] = val;
      }
      else flag = 1;
    }
  }
}  
static int Puzzle(int[] arr) {
  insertionSort(arr);
  return arr[arr.Length - 1];
}
````



#### Submission four (right answer, wrong algorithm)

Now let's say that the student simply used C#'s built-in `array.Max()` function, without performing any sorting:

```c# 
  static int Puzzle(int[] arr) {
    return arr.Max();
  }
```



#### Submission five (right answer, no implementation)

Now let's say that a student doesn't implement their own sorting algorithm, but instead uses C\#'s built-in `Array.Sort()` function (which uses quick sort under the hood, [see documentation](https://docs.microsoft.com/en-us/dotnet/api/system.array.sort?redirectedfrom=MSDN&view=netframework-4.8#System_Array_Sort_System_Array_)). 

```c#	
static int Puzzle(int[] arr) {
  Array.Sort(arr);
  return arr[arr.Length - 1];
}
```



#### Submission six (wrong answer, buggy implementation)

Now let's say that another student submitted a [slightly] incorrect implementation of the quick sort algorithm:

```c# 
static public int Partition(int[] arr, int left, int right) {
  int pivot;
  pivot = arr[left];
  while (true) {
      while (arr[left] <= pivot)  // IndexOutOfRangeException
          left++;
      while (arr[right] >= pivot) // IndexOutOfRangeException
          right--;
      if (left < right) {
          int temp = arr[right];
          arr[right] = arr[left];
          arr[left] = temp;
      } else 
          return right;
  }
}
static public void quickSort(int[] arr, int left, int right) {
  int pivot;
  if (left < right) {
      pivot = Partition(arr, left, right);
      if (pivot > 1) 
          quickSort(arr, left, pivot); 	// Stack overflow
      if (pivot < right) 								// Stack overflow
          quickSort(arr, pivot, right); // Stack overflow
  }
}
// Overall error: IndexOutOfRangeException
static int Puzzle(int[] arr) {
  quickSort(arr, 0, arr.Length - 1); 
  return arr[arr.Length-1];
}
```





## Use Cases

### 1. Specialized feedback based on strategy

In this use case, we would want to give each submission different feedback because they don't all commit the same mistakes. 

For submission **one**, we'd give the feedback: "Correct answer."

For submissions **two** and **three**, we'd give the feedback: "Your algorithm returned the correct answer, but has average time complexity O(n^2), which doesn't match quick sort's O(nlogn). You may have implemented selection/bubble/insertion sort. Instead, consider adding a helper function and the divide-and-conquer recursive strategy."

For submission **four**, we'd give the feedback: "Your algorithm returned the correct answer, but has average time complexity O(n), which doesn't match quick sort's O(nlogn). You did not implement any sorting algorithm."

For submission **five**, we'd give the feedback: "Your algorithm returned the correct answer and has the same average time complexity as quick sort's O(nlogn), but you used C#'s built-in `Sort()` function instead of implementing your own."

For submission **six**, we'd give the feedback: "Your almost there! Your algorithm yields runtime errors (IndexOutOfRangeException), but uses the same strategy as the correct solution. Check the bounds of your for/while loops as well as your recursive calls for missing '+1' or '-1's."

We could use the [Lizard code complexity analyzer](http://www.lizard.ws/#) to estimate how 'complex' code *looks* (as opposed to its actual time complexity). We would calculate the average CCN (Cyclomatic complexity number) for all submissions per cluster, which would allow us to give such personalized feedback.



### 2. Partial credit/penalization*

In this use case, the instructor would use the clusters to give partial credit to submissions based on their adherence to the assignment prompt. 

Submission **one**, we'd give the feedback: "Correct answer."

Submissions **two** and **three** would likely receive the same score, since they all implemented a sorting algorithm, albeit the incorrect one, and produced the correct answer.

Submissions **four** and **five** would be penalized more than submissions two and three, and would likely receive the same score, since they produced the correct answer but did not *implement* a sorting algorithm (they instead used C\#'s built-in functions).

Submission **six** would get penalized equally or less than submissions two and three because, although the correct answer was not reached, the algorithm attempted was quick sort and the submission was close to the correct solution. 

We would calculate the average CCN (Cyclomatic complexity number) via [Lizard](http://www.lizard.ws/) for all submissions per cluster. This is a way of assigning meaning to each cluster, which would help the instructor more quickly understand the contents of each cluster and assign grades.



### 3. Match incorrect submissions to correct ones

In this use case, the tool would match incorrect submissions to correct ones that are similar in approach, in order to provide feedback. This feedback would help students arrive at the correct answer. The feedback given could be a diff between the correct and incorrect submissions. 



 ### 4. Track how student submissions move through different clusters*

In this use case, we could see how students' code changes when they get partial credit on CodeHunt. For example, on CodeHunt, students can get the answer correct and **not** get 3/3 points (it's possible to get 2/3 for having too many lines of code). So we can track student submissions and see how they move throughout the different clusters (i.e., different approaches). This type of analytics may be useful to an instructor to understand how their students are approaching different problems, and how they progress towards the correct answer.



Overall, the research questions we'd like to answer in our use cases are as follow:

###### Research Questions

1. Does our approach cluster submissions by algorithm implementations with a low false-positive rate?
2. Does clustering by both PCs and RVs produce less false-positives than clustering by just PCs?





## CodeHunt Dataset

|     Problem     | Num. Winning/Total C# Subs. | Num. Compiling Subs. | Num. Winning/Total Java Subs. |
| :-------------: | :-------------------------: | :------------------: | :---------------------------: |
| Sector1-Level4  |           63/1294           |         1036         |            ?/1077             |
| Sector2-Level1  |           42/1495           |  (encounters error)  |            ?/1374             |
| Sector2-Level5  |           44/287            |                      |             ?/247             |
| Sector3- Level1 |           15/102            |                      |             ?/156             |
| Sector3-Level2  |           48/287            |                      |             ?/247             |

<span style='color:red'>TODO:</span> Evaluate the following CodeHunt puzzles w/ >= 1 branches in solution:

- Sector1-Level6
- Sector2-Level1
- Sector2-Level2
- Sector2-Level3
- Sector2-Level4
- Sector2-Level5
- Sector2-Level6
- Sector3-Level1
- Sector3-Level2
- Sector3-Level3
- Sector3-Level5
- Sector3-Level6
- Sector4-Level2
- Sector4-Level3
- Sector4-Level4
- Sector4-Level6



### Problem Descriptions

**Sector1-Level4: **Test if a number is a multiple of another number.

**Sector2-Level1: **Compute average of a list of numbers, rounded to closest integer.

**Sector2-Level5: **Find maximum difference between 2 elements in an array.

**Sector3-Level1: **Filter retaining only values >= threshold (a crude noise filter).

**Sector3-Level2: **Compute sum of n-th and (n-1)st Fibonacci numbers.



## Preliminary Results

### Experiment One

In the first experiment, we cluster the winning submissions using only their path conditions.

| **Problem**    | Num. Clusters | **Num. FP Subs. /Num. Total Winning Subs.** |
| -------------- | ------------- | ------------------------------------------- |
| Sector1-Level4 | 6             | 1(?)/63                                     |
| Sector2-Level1 | 14            | 8/42                                        |
| Sector2-Level5 | 5             | 0/44                                        |
| Sector3-Level1 | 2             | 0/15                                        |
| Sector3-Level2 | 4             | 13*/48                                      |

See [doubts surrounding Sector1-Level4 here](#Questions and Doubts) and [quick peak of Sector2-Level5's results here.](#Quick Peak of Sector2-Level5's Results)



### Experiment Two 

In this experiment, we cluster the winning submissions by both their path conditions and return values.

| **Problem**    | Num. Clusters | Num. FP Subs. /Num. Total Winning Subs. |
| -------------- | ------------- | --------------------------------------- |
| Sector1-Level4 | 10            | 0(?)/63                                 |
| Sector2-Level1 | 14            | 5/42                                    |
| Sector2-Level5 | 5             | 0/44                                    |
| Sector3-Level1 | 2             | 0/15                                    |
| Sector3-Level2 | 5             | 13*/48                                  |

*\* = Those submissions that used a different approach than those they were clustered with (e.g., a O(2^n) recursive Fibonacci implementation clustered with submissions that used an iterative O(n) approach).*

*?* = Doubts concerning this figure.



## Quick Peak of Sector2-Level5's Results

Below we show a sample of the clusters produced by running our pipeline on the winning submissions in Sector2-Level5.

### Cluster Zero

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

### Cluster One

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

### Cluster Two

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

### Cluster Three

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

### Cluster Four 

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





## Cyclomatic Complexity Number (CCN)

See the [Lizard](https://github.com/terryyin/lizard) GitHub repository for tool details. One thing to note is that when running Lizard on a given folder, it only analyzes the files with unique (non-duplicate) contents.



Also see [Code Complete](https://www.microsoftpressstore.com/store/code-complete-9780735619678) (p. 458) by McConnell to understand how to interpret CCN values. 

![img](https://i0.wp.com/blog.feabhas.com/wp-content/uploads/2018/07/table2.png?resize=640%2C170&ssl=1)





## Questions and Doubts

I do have doubts with the way that we're counting up the number false-positives. Let's take Sector1-Level4 as an example. Experiment one grouped the following four submissions into the same cluster, whereas experiment two separated them all into different clusters.

Submission one:

``` c#
// User015-1
using System;
public class Program {
  public static bool Puzzle(int x, int y) {
    if(x%y==0) return true;
    return false;
  }
}
```

Submission two:

``` c#
// User034-1
using System;
public class Program {
  public static bool Puzzle(int x, int y) {
    return (x+y)%y==0;
  }
}
```

Submission three:

``` c#	
// User 101-1
using System;
public class Program {
  public static bool Puzzle(int x, int y) {
    if(((float)x/(float)y)%1==0)
		return true;
		return false;
  }
}
```

Submission four:

``` c#
// User083-1
using System;
public class Program {
  public static bool Puzzle(int x, int y) {
    return (x%y)<=0;
  }
}
```



For experiment one, I considered submission three to be a false-positive because it used a different strategy than the other submissions: it used float division and then the modulo operator to check the remainder of the division. 

My questions are: 

**Should submission three be a FP in experiment one? **

**Should submissions two-four be considered FPs in experiment two? **





## Algorithm Sources

[Rosetta Code](http://www.rosettacode.org/wiki/Category:Sorting_Algorithms)

[Tutorials Point](tutorialspoint.com)

[DSA repo](https://github.com/abdonkov/DSA/tree/master/DSA/DSA/Algorithms/Sorting)

[C\# Algorithms repo](https://github.com/aalhour/C-Sharp-Algorithms/tree/master/Algorithms/Sorting)

[C\# Data Structures and Algorithms Textbook](https://github.com/PacktPublishing/C-Sharp-Data-Structures-and-Algorithms/tree/master/Chapter02/SortingAlgorithms) (*)

[Prof. Weiss from FIU](http://users.cis.fiu.edu/~weiss/cs/Sort.cs) (*)

[C\# Star](https://www.csharpstar.com/csharp-algorithms/) (*)





## Bugs

Skipping <u>Sector1-Level6</u> b/c it's giving issues with parsing unicode characters and `string.Contains()`. <u>Sector2-Level3</u> had issues implementing the `String` methods, which was impacting clustering results...so we'll skip this as well. Also <u>Sector2-Level4</u> was running `cluster.py` for 3 days and still hasn't finish...so I will also skip this one. 