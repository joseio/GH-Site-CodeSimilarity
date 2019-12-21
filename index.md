# <center>CodeSimilarity v. 2</center>

*Updated December 21st, 2019*





# Contents

[Synopsis](#Synopsis)

[Potential Applications of Tool](#Potential Applications of Tool)

[Use Cases](#Use Cases)

[CodeHunt Dataset](#CodeHunt Dataset)

[Preliminary Results](#Preliminary Results)

[Bugs](#Bugs)





## Synopsis:

We ran the entire CodeSimilarity v.2 pipeline on only the "winning" (or correct) submissions of CodeHunt's **Sector 2 Level 5** dataset (where students found maximum difference between elements in a given integer array). By "entire pipeline" we mean:	

1. Collecting all unique, concrete tests resulting from running Pex on all submissions

 	2. Collecting all path conditions (PCs) and symbolic return values (SRVs) from re-running Pex on all submissions, this time using only the unique, concrete tests as seeded inputs
 	3. Clustering submissions by PCs and SRVs, according to Z3





## Potential Applications of Tool:

Let's say that we have a problem where the instructor asks students to write a method, `Puzzle` in C# to find the greatest element in an integer array using the **quick sort** algorithm. The method takes in an integer array. Let's say the student submissions to the problem look like this:



#### Submission one (correct)

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



#### Submission five (right answer, wrong algorithm)

```c#	
static int Puzzle(int[] arr) {
  Array.Sort(arr);
  return arr[arr.Length - 1];
}
```



#### Submission six (incorrect)

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

For submissions **two** and **three**, I'd give the feedback: "Your algorithm returned the correct answer, but has average time complexity O(n^2), which doesn't match quick sort's O(nlogn). You may have implemented selection/bubble/insertion sort. Instead, consider adding a helper function and the divide-and-conquer recursive strategy."

For submission **four**, I'd give the feedback: "Your algorithm returned the correct answer, but has average time complexity O(n), which doesn't match quick sort's O(nlogn). You did not implement any sorting algorithm."

For submission **five**, I'd give the feedback: "Your algorithm returned the correct answer and has the same average time complexity as quick sort's O(nlogn), but you used C#'s built-in `Sort()` function instead of implementing your own."

For submission **six**, I'd give the feedback: "Your almost there! Your algorithm yields runtime errors (IndexOutOfRangeException), but uses the same strategy as the correct solution. Check the bounds of your for/while loops as well as your recursive calls for missing '+1' or '-1's."



### 2. Partial credit/penalization

In this use case, the instructor would use the clusters to give partial credit to submissions based on their adherence to the assignment prompt. 

Submissions **two** and **three** would likely receive the same score, since they all implemented a sorting algorithm, albeit the incorrect one, and produced the correct answer.

Submissions **four** and **five** would be penalized more than submissions two and three, and would likely receive the same score, since they produced the correct answer but did not implement a sorting algorithm.

Submission **six** would get penalized less than submissions two and three because, although the correct answer was not reached, the algorithm attempted was quick sort and the submission was close to the correct solution. 



### 3. Match incorrect submissions to correct ones

In this use case, the tool would match incorrect submissions to correct ones that are similar in approach, in order to provide feedback. This feedback would help students arrive at the correct answer. The feedback given could be a diff between the correct and incorrect submissions. 



 ### 4. Track how student submissions move through different clusters

In this use case, we could see how students' code changes when they get partial credit on CodeHunt. For example, on CodeHunt, students can get the answer correct and **not** get 3/3 points (it's possible to get 2/3 for having too many lines of code). So we can track student submissions and see how they move throughout the different clusters (i.e., different approaches). This type of analytics may be useful to an instructor to understand how their students are approaching different problems, and how they progress towards the correct answer.



Overall, the research questions we'd like to answer in our use cases are as follow:

###### Research Questions

1. Can our approach cluster submissions by algorithm implementations with a low false positive rate?
2. Can our approach cluster more faithfully than AST-based approaches?





## CodeHunt Dataset

|     Problem     | Num. Winning/Total C# Subs. | Num. Compiling C# Subs. | Num. Winning/Total Java Subs. | Num. Compiling Java Subs. |
| :-------------: | :-------------------------: | :---------------------: | :---------------------------: | :-----------------------: |
| Sector2-Level4  |           43/623            |           547           |             ?/558             |             ?             |
| Sector2-Level5  |           44/287            |            ?            |             ?/247             |             ?             |
| Sector3- Level1 |           15/102            |           77            |             ?/156             |             ?             |
| Sector3-Level2  |           48/287            |            ?            |             ?/247             |                           |





## Preliminary Results

### Experiment One 

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



### Experiment Two 

Clustering by PC and RV for Sector3-Level2 yielded exactly the same results as clustering by PC only. It is likely because the input is a single integer, meaning that the PCs and RVs were very short (with little variance). 

​	Example: PC: `return x == 0;`,  RV: `return 0;`

I suspect that clustering by PC and RV will be more effective at pruning false-positives on problems with more complex inputs (e.g., int arrays, strings)

#### Cluster Zero 

**TODO:** Show examples of the potential FPs per cluster



### Experiment Three 

#### Cluster Zero





## Bugs

Skipping Sector1-Level6 b/c it's giving issues with parsing unicode characters and `string.Contains()`. Also Sector2-Level4 was running `cluster.py` for 3 days and still hasn't finish...so I will also skip this one.





