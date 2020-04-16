<title>CodeSimilarity v. 2</title>
# <center>CodeSimilarity v. 2</center>

*Last updated March 19th, 2020*





# Contents

[Synopsis](#synopsis)

[Motivating Example](#motivating-example)

[Potential Applications of Tool](#potential-applications-of-tool)

[Use Cases](#use-cases)

[Experimental Setup](#experimental-setup)

[CodeHunt Dataset](#codehunt-dataset)

[Algorithmic Experiments](#algorithmic-experiments)

[Preliminary Results](#preliminary-results)

[Sample of Results](#quick-peak-of-sector2-level5s-results)

[Cyclomatic Complexity Number (CCN)](#cyclomatic-complexity-number-ccn)

[Questions and Doubts](#questions-and-doubts)

[Algorithm Sources](#algorithm-sources)

[Bugs](#bugs)



**TODO:** 

1. Add that Microsoft's tool sucks in clustering our dataset and it's requirements to have > 20 tokens and > 0.7 metric
   1. This is issue: imagine studentA writes long for-loop and studentB writes short C\# LINQ expression. The former may have many tokens, but the latter may not, therefore rejected by Microsoft's tool, whereas ours would cluster them together



## Synopsis:

We aim to cluster student programming submissions (in the homework/quiz/exam setting) by *strategy*. The strategy taken in a program lies in the way the problem space is partitioned into sub-spaces and how the problem is uniquely addressed within individual sub-spaces ([SemCluster](https://www.cs.purdue.edu/homes/roopsha/papers/semcluster_pldi2019.pdf)). More intuitively, if we had two sorting algorithms, insertion sort and bubble sort, they both have the same time complexity but employ different strategies to sort arrays. What makes the two strategies difference is the sequence in which the array elements (partitions or sub-spaces) are handled and changed. 

To better understand what it means to cluster by strategy, we run our tool on 46 submissions, each one containing different implementations of the 11 sorting algorithms listed in the [Algorithmic Experiments](#algorithmic-experiments) section. Ideally, we want our tool to produce 11 clusters, each one containing one type of sorting algorithm. If the ideal is not obtained, then we'd need to answer the following questions:

- *Why* are two different implementations of the same algorithm not clustered together?
- Is there a difference between "algorithm" and "strategy"?
- Can path conditions be used to capture program strategy?



After reaching this understanding, we can compare our tool's clustering results to Microsoft's Near Duplicate Detector (a syntax-based clustering approach) and Semcluster (a sementic-based clustering approach) on the aforementioned sorting algorithms, CodeHunt, and Pex4Fun datasets.





## Motivating Example:

Let's examine the following student submissions to illustrate why many program clustering techniques fail to cluster programs with similar solution strategies. Then we'll see how we **expect** our technique to cluster the submissions.



##### maxMinA:

```c#
// Used default min(), max() functions
using System;
using System.Linq;
public class Program {
  public static int Puzzle(int[] a) {
    return a.Max()-a.Min();
  }
}
```



##### maxMinB:

```c#	
// Implemented min(), max() themselves
using System;
public class Program {
  public static int Puzzle(int[] a) {
    int min = a[0], max = a[0];
    int i = 0, k = 0;
    while((i + 1) != a.Length) {
      if(a[i + 1] > max) max = a[i + 1];
      i++;
    }
    while((k + 1) != a.Length) {
      if(a[k + 1] < min) min = a[k + 1];
      k++;
    }
    return max - min;
  }
}
```



##### maxMinC: 

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



A good technique would place maxMinA and maxMinB in a different cluster than maxMinC, which performs pairwise comparison between each element of  the array to compute the greatest difference. The maxMinA and maxMinB instead both first identify the largest element in the array, then the smallest element of the array, and then subtract them. 







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



### <span style='color:green'>2. Partial credit/penalization*</span>

In this use case, the instructor would use the clusters to give partial credit to submissions based on their adherence to the assignment prompt. We'll focus on this in the paper.

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





## Experimental Setup

#### Algorithms



#### CodeHunt and Pex4Fun

Below are the steps taken in our evaluation setup in the form of a numbered list:

1. Filter out the Java submissions, keeping only the C\# ones (for Pex compatibility)
2. Select those puzzles whose solutions feature at least one branch (so we see variance among the path conditions)
3. Write PUTs for the selected puzzles to evaluate the student submissions against
4. Invoke Pex on the PUTs 
5. Collect all path conditions (PCs) and symbolic return values (SRVs) resulting from step (4)
6. Collect all unique, concrete tests generated during step (4) and place them into a set
  7. Invoke Pex on the PUTs, this time using only the unique, concrete tests as seeded inputs
  8. Collect all PCs and SRVs resulting from step (7)
  9. Parse the PCs for each submission and pass the parsed text to Z3
  10. Build a Z3 model to perform a pairwise comparison between each submissions' PCs for each seeded input
  11. Cluster those submissions whose PCs are logically equivalent for *every* seeded input, according to the Z3 model



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





## Algorithmic Experiments

### Goal

We ran our tool on 46 submissions, each one containing different implementations of the following 11 sorting algorithms: Binary insertion sort (3), bogo sort (5), buble sort (5), cocktail sort (5), cycle sort (4), heap sort (4), insertion sort (4), merge sort (4), pancake sort (3), quick sort (3), selection sort (3), shell sort (3).

The ideal outcome of this experiment is to produce 11 clusters (i.e., one for each algorithm), each one containing only one type of sorting algorithm. 

### Results

After running our tool, 39 clusters were produced. Here's a closer look into those clusters produced:![image-20200224143639294](images\image-20200224143639294.png)

Of course, these results are less than ideal. To understand *why* more submissions implementing the same algorithm were not clustered with each other, we dug deeper. We partitioned the input space, selected one or two representative submissions from each cluster, recorded the executed LOC, array changes, and path conditions for each input partition. Again, this analysis helped us determine why/why not submissions should be clustered together based on SemCluster's definition of *strategy*. 

We divided the input space into the following partitions: 

* Empty int array
* Int array of length = 1
* Sorted ascending int array of length > 2
* Sorted descending int array of length > 2
* Sorted int array of length > 2 containing only elements of equal value



After careful examination of the PCs, we found that the two bubble sorts (which were placed into different clusters) actually yielded equivalent PCs for every input partition. If that's the case, then **why weren't they clustered together?** We found that for the PCs where Pex created variables (e.g., `bool s0 = a[0L] < a[4L]`), Z3 was not associating the variable names (i.e., `s0`) with their corresponding values (i.e., `a[0L] < a[4L]`). This meant that two logically equivalent path conditions wouldn't be clustered if one used variables and the other did not. An example of such a case is seen in the image below:

![image-20200224172323301](images\image-20200224172323301.png)

To resolve this, I first created a function to replace each variable with the underlying values that they hold via recursion. For instance, the last line of this PC would instead read: `&& (a[4L] < a[0L]) && !(a[0L < a[4L]])`. The function works as expected, but encounters memory problems when the number of variables to replace is too big (i.e., over 100). The sheer number of nested variables (e.g., `int s5 = s3 + s17; int s4 = s2;...int s3 = 1; int s4 = 2;`) causes the function to hang and eventually throw memory errors. 

To circumvent this, we only invoke this recursive function on PCs with 100 or less variables and re-run our tool on the dataset. The results are as follow:

![image-20200224201843743](images\image-20200224201843743.png)



**Update (02/24/2020):** 

Zirui just edited the parser to force Z3 into mapping the variables to their corresponding expressions in a more elegant way than my recursive function. Because of this fix, we no longer encounter the aforementioned memory error. 

**Update (02/26/2020):**

We decided to restrict the length of the input arrays from 2 to 4 (inclusive) because for for those inefficient algorithms (i.e., bogo and pancake sort), Pex generated a large number of intermediate vars (i.e., 50 to 200) on those array inputs of length >= 5. So when I passed those PCs to Z3, it seemed to be struggling to map these variables to their underlying expressions, namely in cases where you have some pc of the form:

`bool s8 = a[2] > 2; bool s9 = !s8; bool s10 = s9? s7 : False; bool...`
When I say "struggling" I mean that it truncated the results of the string to z3 object conversion when it printed to terminal. Zirui actually let me know, though, that though it truncates in the terminal, that Z3 actually maintains the mapping internally...so restricting the limit to < 5 is no longer necessary. 

The following figure shows the results of our clustering technique. 31 clusters were formed. Here, two submissions are clustered if, for every valid concrete input, 

![image-20200303163019171](images\image-20200303163019171.png)



#### A. Relaxing our Tool's Clustering Constraints (>= 70% of PCs Match)

Instead of clustering two submissions if and only if their PCs match for every valid concrete input (i.e., those that don't yield a PC of `expression too big`), we tried clustering if their PCs match for at least 70% of such inputs.We ran this experiment with our input arrays restricted to 2 <= length <= 4 and -10 <= array element value <= 10 to keep the runtime quick so that we can more quickly analyze the results. In this experiment, 23 total clusters were formed. The results are seen below:

![image-20200309165740134](images\image-20200309165740134.png)



#### B. Relaxing our Tool's Clustering Constraints (>= 70% of Predicates Match)

Instead of clustering two submissions if and only their two PCs are proven equivalent by Z3 (i.e., Z3 deems that each of their predicates are equivalent), we tried clustering them if at least 70% of their predicates are proven equivalent by Z3. We ran this experiment with our input arrays restricted to 2 <= length <= 4 and -10 <= array element value <= 10 to keep the runtime quick so that we can more quickly analyze the results. In this experiment, 24 clusters were produced. The results are shown below: 

![image-20200309165700033](images\image-20200309165700033.png)

You'll see that the difference between between the results from experiments B and C is that the former clusters two binary insertion sort algorithm implementations, whereas the latter does not cluster any binary insertion sort implementations together. 

#### C. Relaxing our Tool's Clustering Constraints (>= 60% of Predicates Match)

We relaxed the constraints even more to see how considering two PCs to be equivalent if at least 60% of their predicates matched would affect our results. Again, we ran this experiment with our input arrays restricted to 2 <= length <= 4 and -10 <= array element value <= 10 to keep the runtime quick so that we can more quickly analyze the results. In this experiment, 16 clusters were produced. The results are shown below:

![image-20200309165625678](images\image-20200309165625678.png)

Some differences spotted between experiment C's results and the results from the previous experiments include: 

* Diff. b/t this and >=70% **PC** clustering is the underlined 2 clustered binary insertion sorts. This >=60% preds clusters them together whereas the other does not cluster any binary insertion sort implementations together
* Diff. b/t this and >= 70% preds is that this one clusters 4 insertion and 3 selection, whereas the latter clusters only 3 insertion and 2 selection sorts together
* Diff b/t this and >= 70% preds is underlined 4 clustered cocktail sorts. The latter only clusters 3 cocktails together
* Diff b/t this and >= 70% preds is underlined 2 pancake sorts. This one clusters only 2, whereas the latter clusters 3 together

#### D. Relaxing our Tool's Clustering Constraints (>= 70% of Predicates and >= 70% of PCs Match)

We ran another experiment that clusters two submissions if at least 70% of the PC's predicates match for at least 70% of the PCs over all valid, concrete inputs. We, again, ran this experiment with our input arrays restricted to 2 <= length <= 4 and -10 <= array element value <= 10 to keep the runtime quick so that we can more quickly analyze the results. In this experiment, 18 clusters were produced. The results are shown below:

![image-20200309165548622](images\image-20200309165548622.png)



#### E. Relaxing our Tool's Clustering Constraints (>= 80% of Predicates and >= 80% of PCs Match)

We ran another experiment that clusters two submissions if at least 80% of the PC's predicates match for at least 80% of the PCs over all valid, concrete inputs. We, again, ran this experiment with our input arrays restricted to 2 <= length <= 4 and -10 <= array element value <= 10 to keep the runtime quick so that we can more quickly analyze the results. In this experiment, 25 clusters were produced. The results are shown below:

![image-20200309165456621](images\image-20200309165456621.png)



#### F. Comparing our Tool's Results to the Microsoft Near-Duplicate Detector's results

We compared our tool's ability to cluster by strategy to [Microsoft's syntax-based near-duplicate detector](https://github.com/microsoft/near-duplicate-code-detector), and the results of how their tool performed on our dataset of algorithms is shown below:

![image-20200304224449211](images\image-20200304224449211.png)



We see in the above results that Microsoft's tool created four clusters, one of which is "pure" (i.e., comprised of one type of sorting algorithm). Some limitations of Microsoft's near-duplicate detector is that it requires the Jaccard similarity threshold to be at least 0.8 for token-sets and and 0.7 for token multi-sets; it also only runs the tool on submissions that have at least 20 tokens, as highlighted in section three of the [corresponding paper](https://arxiv.org/pdf/1812.06469.pdf). In other words, files with fewer than 20 identifier tokens are not considered duplicates and are excluded from their analysis.



### Analyzing results

Let's look at (E) above and see why all but one insertion sort implementations were clustered together. 



For more analysis of the path condition differences across different implementations [here](https://docs.google.com/presentation/d/1BPeD3sHjuqZ2z7aUnk-uZrJ79KcTVSZMu2gRvk0DYX0/edit?usp=sharing).



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



### Experiment Two a[0L] < a[4L]

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



March 19th, 2020:

One merge sort (User007-attempt001) and two binary insertion sorts (User000-attempt000, User000-attempt001) had `Array.Copy`. This gives Pex problems with tracking because it cannot map the result of `Array.Copy()` back to the original input array. To remedy this, I tried replacing the `Array.Copy()` with traditional for-loops. This was taking too long to debug, so I decided to not consider these three implementations in the approach for now.

April 15th, 2020:

Skipping <u>Sector3-Level5</u> because the puzzles return `int[][]`, which Pex cannot implicitly cast to a Path Condition string. I'm sure a clever workaround could be fashioned, but I will not spend time on that at this time.



## Bugs Fixed

March 18th, 2020:

Found a bug where preambles with `&&` operators were getting split up, which affected our clustering results. Here's an example of a preamble that would cause issues:

PC1 = ` bool s0 = a[0] > a[1]; bool s1 = s0 && a[2] > a[1]; return 5`

The above expression for`s1` would get split into two terms, which in many cases is undesirable. Fixed this bug by not splitting up preamble expressions.