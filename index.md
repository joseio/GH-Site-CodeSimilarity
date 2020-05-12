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

[Taxonomy](#taxonomy)

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
3. Write PUT wrappers for the selected puzzles to evaluate the student submissions against
4. Invoke Pex on the PUT wrappers 
5. Collect all path conditions (PCs) and symbolic return values (SRVs) resulting from step (4)
6. Collect all unique, concrete tests generated during step (4) and place them into a set
  7. Invoke Pex on the PUTs, this time using only the unique, concrete tests as seeded inputs
  8. Collect all PCs and SRVs resulting from step (7)
  9. Parse the PCs for each submission and pass the parsed text to Z3
  10. Build a Z3 model to perform a pairwise comparison between each submissions' PCs for each seeded input
  11. Cluster those submissions whose PCs are logically equivalent for *every* seeded input, according to the Z3 model



## CodeHunt Dataset

|     Problem     | Num. Winning/Total C# Subs. | Num. Winning/Total Java Subs. |
| :-------------: | :-------------------------: | :---------------------------: |
| Sector2-Level1  |           42/1495           |            15/1374            |
| Sector2-Level2  |           48/767            |            79/739             |
| Sector2-Level5  |           44/287            |            55/247             |
| Sector2-Level6  |           55/249            |            48/334             |
| Sector3- Level1 |           15/102            |            22/156             |
| Sector3-Level2  |           48/259            |            51/207             |
| Sector3-Level6  |           32/178            |            18/229             |
| Sector4-Level2  |           10/128            |             7/126             |
| Sector4-Level3  |            4/72             |             5/125             |
| Sector4-Level4  |            8/732            |            12/427             |
| Sector4-Level6  |            10/61            |            15/163             |



### CodeHunt Problem Descriptions

All of the CodeHunt puzzles evaluated have clear problem statements and are listed below:

- Sector2-Level1: Compute average of a list of numbers, rounded to closest integer
- Sector2-Level2: Count the depth of nesting parentheses in a string
- Sector2-Level5: Find maximum difference between 2 elements in an array
- Sector2-Level6: Generate the string of binary digits for n
- Sector3-Level1: Filter retaining only values >= threshold -- a crude noise filter
- Sector3-Level2: Compute sum of n-th and n-1st Fibonacci numbers
- Sector3-Level6: Compute the set difference of a\b
- Sector4-Level2: Compute n choose m, i.e. n!/(m! * (n-m)!)
- Sector4-Level3: Given int array inputs a and f , apply b[i] = f[a[i]] and return b
- Sector4-Level4: Return the (first) value with the most number of 1's in its binary representation
- Sector4-Level6: Advance each character in a string by the Fibonacci number evaluated at that character's integer ASCII value
- Sector9-Level1 (Algorithms dataset): Implement a sorting algorithm to sort an integer input array in ascending order



### Pex4Fun Dataset

*Note:* Dataset in paper

Below are the descriptions of the puzzles that explicitly featured problem statements in them, along with the problem statements that I inferred by examining student submissions and instructor solution:

* 33: Use a while loop to count the number of digits in the input number x
* 34: (inferred) Given an integer n as input, return a string of all numbers leading up to and including n, i.e., in range [0, n]
* 35: Given an integer as input, return a string of multiples of the input that are less than the input squared
* 36: (inferred) Given an integer as input, return a string of all numbers from [0, n) three times
* 37: Display the results of integer division
* 38: Given an input integer n and string, print the string n times
* 39: Print the letters in the English alphabet
* 40: Counting the number of occurrences of the letter "a" in the input string
* 42: (inferred) Given an integer as input, return a string of multiples of the input that are less than or equal to the input squared, i.e., [0, 1, 4, 9, ...n^2]
* 43: (inferred) Given an integer as input, return a string of multiples of the input that are less than or equal to the input squared in reverse order, i.e., [n^2, ...9, 4, 1, 0]
* 45: (inferred) Given input integer, calculate its factorial
* 47: Sum the even-indexed numbers in the input array
* 48: (inferred) Given an input integer n, get a list of all integers in reverse order (i.e., [n, 1]), then multiply the index by the element value and add it to a running sum (e.g., `sum += a[i] * (i+1)` )
* 49: (inferred) Given an input string, return a "_" for every letter in the input string (e.g., "you" -> "\_ _ _")
* 50: (inferred) Given an input string, implement Caesar's cipher with offset of 5 and return the encoded string
* 53: (inferred) Given an input integer, return true if it's less than 50, else return false
* 55: (inferred) Given an input integer, return "zero" if it's 0, "negative" if it's negative, and "positive" if it's positive
* 55: (inferred) Given an input integer that is less than 1000, return the number of digits the integer has
* 58: (inferred)  Given an input integer, return "even" if the number is even and "odd" otherwise
* 59: (inferred)  Given an input integer, return "multiple of 5" if it's a multiple of 5 and "not a multiple of 5" otherwise
* 60: (inferred)  Given two input integers `i` and `x`, return "multiple of {x}" if is a multiple of x, and "not a multiple of {x}" otherwise 
* 61: (inferred) Given three input integers `i`, `j`, and `k`, return "yes" if i = 2j and j = 2k, else return "no"
* 62: (inferred) Given an input integer, return 0 if it's in range [0, 7], else return 7 if it's in range (7, 14], else return 21 if it's greater than 14
* 63: (inferred) Given an input string, return "short" if its length is less than 4, else "average" if its length is in range [4, 8), else "long" if it's in range [8, 15)
* 64: Check if an input integer represents a "fancy year," a year whose digits have all the same number
* 65: Given three sides of a shape, figure out which two sides add to equal the remaining side squared
* 73: (inferred) Given two input integers `i` and `j`, return true if i^2 == j, and false otherwise
* 74: (inferred) Given an input string, return "yes" if it contains all lowercase letters, else false
* 75: (inferred) Given an input string, make every letter at an even index uppercase, and every letter at odd index lowercase
* 83: (inferred) Given two input strings, return the length of the longer string
* 105: (inferred) Given an input integer array list `list`and an integer `i`, return true if `i` is in `list`, else false
* 107: (inferred) Given an input integer `n`, return an integer array containing all integers in range [0, n]
* 110: (inferred) Given an input integer, return an integer array list containing all integers in range [0, n] in reverse order
* 111: (inferred) Given an input integer array, negate each element in-place and return the edited array
* 112: (inferred) Given an input integer array, return the reversed array
* 132: (inferred) Given an input integer array `numbers` and an integer `x`, return the number of occurrences of `x` in `numbers`
* 135:  Find the last index of x in an input array
* 140: (inferred) Given an input integer array, return true if it's sorted in ascending order, else false
* 141: (inferred) Given an input string array, return true if it's sorted in ascending alphabetical order, else false
* 143: (inferred) Given an input string array, return the array sorted in alphabetical ascending order
* 144: (inferred) Given an input string, implement Caesar's cipher with offset of 7 and return the encoded string
* 152: (inferred) Given an input string, return true if it's a palindrome, else false

An interesting thing I noticed in throughout the Pex4Fun dataset is that sometimes the instructions explicitly state how the students should solve the problems (e.g., with or without a while-loop), but many students disregard these instructions and still end up getting the answer correct (e.g., puzzle 33). 



#### Puzzles to be Discarded

We examined the  instructor solutions for each of the Pex4Fun puzzles to find those with unclear problem statements. We also examined the student submissions to find any puzzles for which they blatantly tried to guess and check to arrive at the right answer, in which case we would remove that puzzles from our evaluation, since this behavior is not representative of the typical CS education classroom. The puzzles in question are as follows:

* Also, instructions explicitly say to use "while-loop," but many students didn't do so and still got answer correct

* 37: Students (i.e., "rockrose") just guesses and checks...not representative
* 39: Students (i.e., "Sadlic") just guesses and checks...not representative
* 56: Many students did guess and check, also our evaluation subjects may contain incorrect submissions, since we assume that each student's last submission is correct...not always true, which is apparent in this puzzle
* 106: Many of the winning answers don't even involve a branch



### Peking University Dataset

#### Problem Descriptions

The problems from the PKU dataset didn't come with any problem descriptions, so I inferred them by manually inspecting the student submissions

* Homework 1: (inferred) Given an input integer n, find and count all positive integers in range [0, n] that (1) share same first and last digits (e.g., 11, 101, 161) and (2) are not divisible by any preceding odd number

* Homework 2: (inferred) Given input integer n, find the smallest value for x that makes the following equation true: $(Σ (1/j), j=1, x) > n$

* Homework 3: (inferred) Given an input integer and an array of doubles, iterate through each array element and keep a running sum of the element * factor, where factor is: 

  * 0 if element < 100
  * 0.1 if element in range [100, 200)
  * 0.3 if element in range [200, 500)
  * 0.5 if element >= 500

  Then return the sum.



#### Submissions to be Discarded

Many of the correct student submissions failed to compile after they were converted from C/C++ to C#. The most common reason for these compilation failures were things like incorrect data types in method signatures, method bodies, and also visibility issues (i.e., public, private, static methods and classes). I cleaned the dataset to fit my needs, such that the subjects that I evaluate are clean and able to be compiled. Here are some stats on how many correct, non-compiling submissions I needed to filter out of each puzzle to perform the above cleaning:

* Hw1: 55 (of a total 118) submissions removed
* Hw2: 111 (of a total 321) submissions removed
* Hw3: 101 (of total 641) submissions removed
* Hw4: 76 (of total 89) submissions removed



Actually, due to the bug (listed in the bugs section at bottom of page), we are removing Hw4 from our evaluation, since it caused our clustering algorithm to hang for way too long.



**Important:** Instead of setting thresholds, I copied the directories and cleaned the submissions such that they no longer contain the duplicate class. So then I re-ran dupFinder and Near Dupe on these cleaned submissions. Note: In tables in paper, "6x2" means that there were six clusters produced by the syntactic tool, each having two submissions inside.



\* *Note:* 'o' = our tool, 'd' = dupFinder, 'n' = near dupe

| Assignment | Cluster 1 (o/d/n) | Cluster 2 (o/d/n) | Cluster 3 (o/d/n) | Cluster 4 (o/d/n) | Cluster 5 (o/d/n) |
| :--------: | :---------------: | :---------------: | :---------------: | :---------------: | :---------------: |
| Homework 1 |      62/55/0      |       1/2/0       |       0/0/0       |       0/0/0       |       0/0/0       |
| Homework 2 |      204/0/0      |       6/0/0       |       0/0/0       |       0/0/0       |       0/0/0       |
| Homework 3 |      517/0/0      |       9/0/0       |       0/0/0       |       0/0/0       |       0/0/0       |



## Algorithmic Experiments

### Dataset

First, we visited [RosettaCode](http://www.rosettacode.org/wiki/Category:Sorting_Algorithms) to get a list of sorting algorithms (which contains comprehensive list of them). Then we searched each one on Google for C# implementations for the first three pages of results. We selected the algorithms that we were able to get more than five visually different C# implementations for. Sorting algorithms on RosettaCode:

\* Note:  ✓ = included in dataset, ✗ = not included in dataset

- Bead sort ([1](https://www.programmingalgorithms.com/algorithm/bead-sort/)) ✗


- Binary Insertion Sort...not in Rosetta Code...remove from dataset ✗
- Bogosort ([1](https://rosettacode.org/wiki/Sorting_algorithms/Bogosort#C.23), [2](https://github.com/mariansam/Bogosort.NET/blob/master/Bogosort/Bogosort.cs), [3](http://www.siafoo.net/snippet/213), [4](https://www.programmingalgorithms.com/algorithm/bogo-sort/), [5](https://programm.top/en/c-sharp/algorithm/array-sort/bogosort/))...✗ <-- I need to delete this from dataset... ✗
  - To remove dupes: 2
- **Bubble sort** ([1](https://codereview.stackexchange.com/questions/142720/bubble-sort-in-c), [2](http://www.rosettacode.org/wiki/Sorting_algorithms/Bubble_sort#C.23), [3](https://www.csharpstar.com/csharp-program-to-perform-bubble-sort/), [4](https://www.geeksforgeeks.org/bubble-sort/), [5](https://github.com/abdonkov/DSA/blob/master/DSA/DSA/Algorithms/Sorting/BubbleSorter.cs), [6](https://en.wikibooks.org/wiki/Algorithm_Implementation/Sorting/Bubble_sort#C#)) ✓
- Circle Sort (nothing) ✗
- Cocktail sort ([1](https://exceptionnotfound.net/cocktail-shaker-sort-csharp-the-sorting-algorithm-family-reunion/), [2](http://www.rosettacode.org/wiki/Sorting_algorithms/Cocktail_sort#C.23), [3](https://www.geeksforgeeks.org/cocktail-sort/), [4](https://www.growingwiththeweb.com/2016/04/cocktail-sort.html), [5](https://www.programmingalgorithms.com/algorithm/cocktail-shaker-sort/)) <-- I need to delete this from dataset... ✗
- Comb sort  ([1](https://buffered.io/posts/sorting-algorithms-the-comb-sort/), [2](https://gist.github.com/PhearTheCeal/4643988), [3](https://rosettacode.org/wiki/Sorting_algorithms/Comb_sort#C.23), [4](https://www.geeksforgeeks.org/comb-sort/)) ✗
- Counting sort  ([1](http://algorithmsandstuff.blogspot.com/2014/07/counting-sort-in-c-sharp.html), [2](https://www.geeksforgeeks.org/counting-sort/), [3](https://rosettacode.org/wiki/Sorting_algorithms/Counting_sort, [4](http://www.programming-algorithms.net/article/40549/Counting-sort), [6](https://www.alphacodingskills.com/cs/pages/cs-program-for-counting-sort.php))  ✗
- Cycle sort ([1](https://riptutorial.com/algorithm/example/24968/csharp-implementation), [2](http://www.algostructure.com/sorting/cyclesort.php), [3](https://github.com/aalhour/C-Sharp-Algorithms/blob/master/Algorithms/Sorting/CycleSorter.cs#L44), [4](https://www.geeksforgeeks.org/cycle-sort/))  <-- I need to delete this from dataset... ✗
- Gnome sort ([1](https://www.programmingalgorithms.com/algorithm/gnome-sort/), [2](https://exceptionnotfound.net/gnome-sort-csharp-the-sorting-algorithm-family-reunion/), [3](https://rosettacode.org/wiki/Sorting_algorithms/Gnome_sort#C.23), [4](https://programm.top/en/c-sharp/algorithm/array-sort/gnome-sort/)) ✗
- **Heapsort** ([1](https://github.com/abdonkov/DSA/blob/master/DSA/DSA/Algorithms/Sorting/HeapSorter.cs), [2](http://www.rosettacode.org/wiki/Sorting_algorithms/Heapsort#C.23), [3](http://users.cis.fiu.edu/~weiss/cs/Sort.cs), [4](https://github.com/aalhour/C-Sharp-Algorithms/blob/master/Algorithms/Sorting/HeapSorter.cs), [5](https://www.tutorialspoint.com/heap-sort-in-chash), [6](https://www.growingwiththeweb.com/2012/11/algorithm-heapsort.html)) ✓
- **Insertion sort** ([1](https://github.com/abdonkov/DSA/blob/master/DSA/DSA/Algorithms/Sorting/InsertionSorter.cs), [2](http://www.rosettacode.org/wiki/Sorting_algorithms/Insertion_sort#C.23), [3](https://github.com/aalhour/C-Sharp-Algorithms/blob/master/Algorithms/Sorting/InsertionSorter.cs), [4](https://github.com/PacktPublishing/C-Sharp-Data-Structures-and-Algorithms/blob/master/Chapter02/SortingAlgorithms/InsertionSort.cs), [5](https://www.tutorialspoint.com/insertion-sort-in-chash), [6](https://www.w3resource.com/csharp-exercises/searching-and-sorting-algorithm/searching-and-sorting-algorithm-exercise-6.php)) ✓
- **Merge sort** ([1](https://github.com/abdonkov/DSA/blob/master/DSA/DSA/Algorithms/Sorting/MergeSorter.IList.cs), [2](https://programm.top/en/c-sharp/algorithm/array-sort/merge-sort/), [3](https://www.w3resource.com/csharp-exercises/searching-and-sorting-algorithm/searching-and-sorting-algorithm-exercise-7.php), [4](http://users.cis.fiu.edu/~weiss/cs/Sort.cs), [5](https://www.c-sharpcorner.com/blogs/a-simple-merge-sort-implementation-c-sharp), [6](https://codeburst.io/stupids-guide-to-merge-sorting-algorithm-dfeca6094d7f)) ✓
- Pancake sort ([1](https://www.geeksforgeeks.org/pancake-sorting/), [2](http://www.rosettacode.org/wiki/Sorting_algorithms/Pancake_sort#C.23), [3](https://tutorialspoint.dev/algorithm/sorting-algorithms/pancake-sorting))   <-- I need to delete this from dataset... ✗
- Patience sort (nothing) ✗
- Permutation sort (aka BogoSort) ✗
- **Quicksort** ([1](https://github.com/abdonkov/DSA/blob/master/DSA/DSA/Algorithms/Sorting/QuickSorter.cs), [2](http://www.rosettacode.org/wiki/Sorting_algorithms/Quicksort#C.23), [3](https://github.com/aalhour/C-Sharp-Algorithms/blob/master/Algorithms/Sorting/QuickSorter.cs))...extra not yet in dataset: [extra1](https://www.w3resource.com/csharp-exercises/searching-and-sorting-algorithm/searching-and-sorting-algorithm-exercise-9.php), [extra2](http://csharpexamples.com/c-quick-sort-algorithm-implementation/), [extra3](https://gist.github.com/lbsong/6833729) ✓
- Radix sort ([1](https://www.geeksforgeeks.org/radix-sort/), [2](https://rosettacode.org/wiki/Sorting_algorithms/Radix_sort#C.23), [3](https://www.codeproject.com/Articles/32382/Radix-Sorting-Implementation-with-C), [4](http://algorithmsandstuff.blogspot.com/2014/06/radix-sort-in-c-sharp.html)) ✗
- Selection sort ([1](https://github.com/aalhour/C-Sharp-Algorithms/blob/master/Algorithms/Sorting/SelectionSorter.cs), [2](https://github.com/PacktPublishing/C-Sharp-Data-Structures-and-Algorithms/blob/master/Chapter02/SortingAlgorithms/SelectionSort.cs), [3](https://www.csharpstar.com/c-program-to-perform-selection-sort/))  <-- I need to delete this from dataset... ✗
- Shell sort ([2](http://www.rosettacode.org/wiki/Sorting_algorithms/Shell_sort#C.23), [3](https://www.tutorialspoint.com/shell-sort-program-in-chash)) ...extra not yet in dataset: [extra1](http://users.cis.fiu.edu/~weiss/cs/Sort.cs), [extra2](https://github.com/aalhour/C-Sharp-Algorithms/blob/master/Algorithms/Sorting/ShellSorter.cs), [extra3](https://github.com/abdonkov/DSA/blob/master/DSA/DSA/Algorithms/Sorting/ShellSorter.cs) <-- I need to delete this from dataset... ✗
- Sleep sort ([1](http://www.rosettacode.org/wiki/Sorting_algorithms/Sleep_sort#C.23)) ✗
- Stooge sort ([1](http://www.rosettacode.org/wiki/Sorting_algorithms/Stooge_sort#C.23)) ✗
- Strand sort (none) ✗
- Tree sort on a linked list (none)



### Goal

We ran our tool on 46 submissions, each one containing different implementations of the following 11 sorting algorithms: Binary insertion sort (3), bogo sort (5), buble sort (5), cocktail sort (5), cycle sort (4), heap sort (4), insertion sort (4), merge sort (4), pancake sort (3), quick sort (3), selection sort (3), shell sort (3).

The ideal outcome of this experiment is to produce 11 clusters (i.e., one for each algorithm), each one containing only one type of sorting algorithm. 

### Results

We narrowed down the algorithms that we're evaluating such that we now have Bubble, heap, insertion, merge, and quick sort, each one having **six** total implementations.

We ran our tool on the algorithms dataset, then ran the JetBrains dupFinder and MIcrosoft CodeClone tools on the equivalence classes that our tool produced and got the following results:

\* *Note:* 'o' = our tool, 'd' = dupFinder, 'n' = near dupe

|   Algorithm    | Cluster 1 (o/d/n) | Cluster 2 (o/d/n) | Cluster 3 (o/d/n) | Cluster 4 (o/d/n) | Cluster 5 (o/d/n) | Cluster 6 (o/d/n) |
| :------------: | :---------------: | :---------------: | :---------------: | :---------------: | :---------------: | :---------------: |
|  Bubble Sort   |       6/1/5       |       0/1/1       |       0/1/0       |       0/1/0       |       0/1/0       |       0/1/0       |
|   Heap Sort    |       1/1/1       |       1/1/1       |       1/1/1       |       1/1/1       |       1/1/1       |       1/1/1       |
| Insertion Sort |       5/1/3       |       1/1/1       |       0/1/1       |       0/1/1       |       0/1/0       |       0/1/0       |
|   Merge Sort   |       1/1/1       |       1/1/1       |       1/1/1       |       3/1/1       |      0 /1/1       |       0/1/1       |
|   Quick Sort   |       1/1/1       |       1/1/        |       1/1/1       |       2/1/1       |       1/1/1       |       0/1/1       |





###### The results below are outdated

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



## Taxonomy

Below, I assign explicit and consistent labels to each cluster after having manually inspecting the submissions inside.

### CodeHunt

**Sector2-Level1:**

Cluster0-

* 1x: Uses `Array.Average()` and `Math.Floor()`. Unconditional round up; strangely adds 1 and floors if average is negative, else adds 0.5 and floors if average positive 
  * ^ Likely **false positive** b/c if test case had negative numbers, this would give incorrect answer

Cluster1-

* 4x: Uses for-loop to accumulate average, then round up +0.5 *iff* decimal >= 0.5 <span style='color:red'>(Group1)</span>
* 1x: Uses `Array.Average()` and then round up +0.5 *iff* decimal >= 0.5 <span style='color:red'>(Group1)</span>
* 2x: Uses for-loop to accumulate average, then uses array length to round up/down accordingly...doesn't take decimals into account <span style='color:blue'>(Group2)</span>
* 1x: Uses `Array.Average()` and `Math.Truncate()`. Rounds up +0.5 *iff* decimal >= 0.5 <span style='color:red'>(Group1)</span>
* 2x: Increment all elements in second half of array by 1, then take average...basically rounds up to nearest int <span style='color:green'>(Group3)</span>
* 2x: Uses for-loop to accumulate average, then `Math.Round()` to round up/down, depending if decimal >= 0.5 or < 0.5, respectively <span style='color:red'>(Group1)</span>
* 1x: Uses for-loop to accumulate average, then rounds up to nearest integer *iff* decimal > **0.7** <span style='color:purple'>(Group4)</span>
* 1x: Uses `Array.Average()`, then adds/subtracts 0.5 based on the sign of the average (i.e., the smart way of doing cluster 0's strategy) <span style='color:orange'>(Group5)</span>

Cluster2- 

* 2x: Returns 0 if `Array.Average()` is negative, else round up if decimal >= 0.5, else round down <span style='color:red'>(Group1)</span>
* 2x: Use `Array.Average()`, then use `Math.Round()` + 1 if average decimal == 0.5, else just return `Math.Round()` <span style='color:blue'>(Group2)</span>
  * ^ Likely **false positive** b/c yields incorrect answer when average has decimal 0.5, ex: {1, 2}. Actual avg = 1.5, but this prog yields 3. Code DOES work on {0, 1}, however, b/c C#'s Round() converts 0.5 down to 0 (but rounds 1.5 up to 2 for some reason). Test cases captured the second case but not the first. (User012-3-attempt050-20140920-215115-winning2, User012-4-attempt052-20140920-215334-winning2).
* 2x: Uses `Array.Sum()`, divides by length, then adds 0.01 before casting to int via `Convert.ToInt32()`. They add 0.01 for same reason as above...C# rounds 0.5 down to 0 but rounds 1.5 up to 2.  <span style='color:green'>(Group3)</span>

Cluster3-

* 2x: Use for-loop to accumulate average, if average is negative integer, then unconditionally add 1, else floor the average (via `Math.Floor()`, then add 0.5, then conv to int) 
  * ^ Likely **false positive** b/c yields incorrect ans when array.Length > 1 and average is negative int, ex: {-1, -1}. Expected = -1, actual = 0. Test cases didn't capture this.

Cluster4- 

* 1x: Use for-loop to accumulate average, then adds 0.1  and returns`Math.Round()`if average decimal == 0.5, or returns 0 if average in range (-0.5, 0), or adds 1 if average is negative non-int and returns `Math.Round()`. <span style='color:red'>(Group1)</span>
* 2x: Uses `Array.Average()`, then `Math.Round(avg+/-0.05)`, where the 0.05 depends on the sign of the average <span style='color:blue'>(Group2)</span>

Cluster5-

* 10x: Use for-loop to accumulate average, then unconditionally adds 1 if array.Length > 1 and average is negative, else adds 0.5 to average and casts result to integer
  * ^ Likely **false positive** b/c yields incorrect answer when array.Length > 1 and average negative, ex: {-1, -1}. Expected = -1, actual = 0

Cluster6- 

* 1x: Use for-loop to accumulate average, then unconditionally adds 1 if average is negative, else if `avg - (int)avg == 0.5` then return `avg` (if avg is negative) and `(int)(avg+0.5)` (if avg is positive), else if average is positive, add 0.5 to it and floor the result

Cluster7-

* 2x: Use for-loop to accumulate average, return 0 is average is negative, else if has decimal == 0.5 then  return`Math.Round(avg) + 1` , else just return `Math.Round(avg)`.
  * ^ Likely **false positive** b/c yields incorrect answer when average has decimal == 0.5, ex: {1, 2}, expected = 2, actual = 3. 

Cluster8-

* 1x: Uses `Array.Average()`, then adds 0.05, then `Math.Round()` the result, then converts to int via `Convert.ToInt32()`. If result of int conversion is positive or `-1` then return the result, else return 0.
  * ^ Likely **false positive** b/c yields incorrect answer for averages that are less than -1, ex: {-2}, expected = -2, actual = 0.

Cluster9-

* 1x: Returns 0 if `Array.Average() <= -1 `  and array.Length > 1, return ceiling of average if `avg - (int)avg >= 0.5`, else just return `(int)avg`.

Cluster10-

* 1x: Use for-loop to accumulate average, then returns 0 if (hard-coded) array[0] == -1 and array[1] == -1, else truncates average if its decimal is in range [0, 0.5), else adds `1` to average and truncates result if decimal in range [0.5, 0.9], else just returns `(int)avg`.



**Sector2-Level1:**

Difference is that cluster 0 returns 0 as soon as count goes negative whereas cluster 1 does not.

Cluster0-

* 27x: Uses for-loop to find '(' , ')' and increments/decrements count, respectively. *Returns 0 as soon as count goes negative* (i.e., more right parens than left)
* 3x: Uses for-loop to push each '()' onto stack. Pops if parens are matched
* 3x: Use while-loop to replace each instance of '()' with empty string, and count the number of times the replacement happens

Cluster1-

* 3x: Uses Linq to `Select` the '()' in string and increments/decrements count, respectively. Returns 0 if count goes negative (i.e., more right parens than left)
* 3x: Uses for-loop to find '(' , ')' and increments/decrements count, respectively. Returns 0 if count goes negative (i.e., more right parens than left)
* 2x: Uses for-loop to push each '()' onto stack. Pops if parens are matched
* 2x: Uses for-loop to find number of left and right parens, returns 0 if unbalanced parens, else uses another for-loop to make sure parens ordering is correct
* 1x: Splits array first by '(', then by ')', then compares lengths of both parts to derive answer
  * ^ Likely **false positive** b/c this strategy is very unique...(User046-1-attempt004-20140920-103858-winning3)



**Sector2-Level5:**

Cluster0-

* 2x: Use `Array.Sort()` then iterates to find biggest difference

Cluster1-

* 19x: Use C# built-in `Array.Min()` and `Array.Max()` funcs
* 11x: implement their own Array.Min() and Array.Max() funcs or use for-loop to find max and min values

Cluster2-

* 10x: Uses `Array.Sort()` then computes (last ele - first ele)

Cluster3- 

* 2x: Implements bubble sort, then computes (last ele - first ele)



**Sector2-Level6:**

Cluster0- 

* 36x: Uses C#'s built-in int to binary converter, `Convert.ToString(x, 2)`
* 4x: Implements their own int to binary method (iterative bit shifting)
* 2x: Recursively performs modulo on x/2 
* 13x: Iteratively performs modulo on x/2



**Sector3-Level1**

Cluster0- 

* 3x: Uses `Select` statement to set all array elements greater than threshold to 0
* 11x: Uses for-loop to set all array elements greater than threshold to 0

Cluster1-

* 1x: Uses for-loop to set all array elements that yield 0 when divided by the threshold to 0 (i.e., `if (a[i]/t == 0)`)



**Sector3-Level2**

Cluster0-

* 26x: Iterative solution w/ for-loop
* 6x: Iterative solution w/ `Enumerable.Range()`, `Skip()`, and `Aggregate()`
* 16x: Recursive solution



**Sector3-Level6**

Cluster0- 

* 2x: Converts both integer arrays into hash sets, computes set difference via `setA.ExceptWith(setB)`, then converts set difference back to array
* 7x: Returns difference between two arrays *without* converting to hash set (i.e., `a.Except(b).ToArray()`)
* 4x: Does group join to combine both integer arrays, then adds the unique elements to a new integer array and returns it

Cluster1- 

* 5x: If array `a` only has one unique element and array `b` contains that unique element, then return empty array, else return `a`
* 5x:  Defines an empty array list and adds all of array `a`'s elements to it via for-loop, then removes from it those elements that are shared with array `b` via for-loop
* 3x: Uses C# Linq expressions (`Any()` or `Where(...Contains())`) to return all distinct elements from a that aren't in b as an array

Cluster2-

* 1x: Defines empty array, then iterates through array `a` w/ for-loop to see if a given element is contained in array `b` (via `Array.IndexOf()`). If it's not, then add the element to the empty array. Then sorts array then returns it.
* 2x: Uses C# Linq expressions (`Where(...IndexOf())`) to return all distinct elements from a that aren't in b as an array

Cluster3-

* 1x: First sorts array `a`. Then defines empty array, then iterates through array `a` w/ for-loop to see if a given element is contained in array `b` (via `Array.IndexOf()`). If it's not, then add the element to the empty array. 
  * ^ Likely **false positive**, b/c same code as a submission in cluster 2, except the `Array.Sort()` statement is in a different place

Cluster4-

* 1x: If array `b` is empty, then iterate through array `a` and return `a` if any of its elements are positive, else return [0]. Else, if neither array is empty, then iterate through `b` and see if any of `b`'s elements are equal to `a[0]`. If this condition evaluates to true, then return [0], else return `a`.

Cluster5-

* If both arrays are empty, return [0]. If they're both of length 1 and have same element, return [0] as well. If the sum of `a`'s elements  0 and b is empty, then return [0]. Else, copy the elements of `a` into a new array (via for-loop) and return new array.

**Sector4-Level2**

Cluster0- 

* 1x: Uses C# Linq expressions (`Range().Aggregate()`) to calculate each factorial, then combines them all into a single equation
* 3x: Uses recursion to calculate factorials in following "addition" format: `subset(n-1, k-1) + subset(n-1, k)`

Cluster1-

* 1x: Uses recursion in the following *traditional* equation format: `f(m)/(f(n)*f(m-n))`

Cluster2-

* 3x: Creates inline, recursive factorial func that follows traditional equation format: `f(m)/(f(n)*f(m-n))`
  * ^ Likely **false positive** b/c this strategy is the same as the one in cluster 1, except this uses inline func, which may not be getting interpreted by Pex properly

Cluster3- 

* 1x: Iterative solution (optimized)

Cluster4- 

* 1x: Iterative solution (un-optimized)
  * ^ Likely **false positive** b/c this is a less efficient version of that in cluster 3. Cluster 3 adds *one single line* that decreases the number of iterations that need to be run...they both yield the same results though (they allegedly diverge on {7, 6}, but they yield same output here...it's just that cluster 3's has less iterations)

**Sector4-Level3**

Cluster0-

* 3x: Uses for-loop to perform `b[i] = f[a[i]]`, given input arrays `a` and `f`
* 1x: Uses C# Linq expressions (`Range().Select()`) to perform `b[i] = f[a[i]]`

**Sector4-Level4**

Cluster0-

* 6x: Nested for-loop where they count the number of factors each array element has; the element with the most factors is returned

Cluster1- 

* 1x: In a while-loop they record the number of times an array element can be bitwise AND'd w/ the element minus one (i.e., `x &= (x-1)`) until the element equals 0; the element w/ most iterations is returned

Cluster2- 

* 1x: Does some hardcoding to return certain values if the array length is three or if the number of non-zero elements in the array is less than two. Otherwise, they calculate the GCD for each element and divide the element by it. The element with the highest quotient is returned.

**Sector4-Level6**

Cluster0-

* 7x: Iteratively calculates Fibonacci sequence and shifts each character per iteration, uses `%26` to wrap the shifted character back to beginning of alphabet so it doesn't exceed ['a'-'z'] boundaries
* 1x: Uses C#'s Linq expressions (`Range().Select()`)  to first store Fibonacci into array, then another set of similar LInq expressions to shift each character by the values stored in the Fibonacci sequence array
* 1x: Uses recursion to calculate Fibonacci sequence and Linq expressions (`Range().Select()`)  to shift each character by the sequence
  * ^ Likely **false positive**, shouldn't we be able to distinguish iterative from recursive?

Cluster1-

* 1x: Iteratively calculates Fibonacci sequence and shifts each character per iteration, does *not* use modulo operator, instead manually subtracts 26 whenever shifted character outside of ['a'-'z'] range
  * ^ Likely **false positive** b/c the strategy here is the same as that in cluster 0, except it doesn't use modulo operator





### Algorithms

**Bubble Sort**

Cluster0-

* 3x: Nested for-loop
* 3x: Nested for-loop with short-circuit optimization

**Heap Sort**

See [this source](https://www.reddit.com/r/algorithms/comments/8n543e/what_is_the_difference_between_maxheapify_and/) and [this source](https://en.wikipedia.org/wiki/Heapsort#Floyd's_heap_construction) for different heap constructions. I'm not too well versed on the nuanced differences between these different heap constructions yet, but when I tested the six implementations below on the int input [9, 6, 3, 12], I made the following observations:

**I'm not confident on the below...can I get any deeper than this???**

Cluster0-

* 1x: Sift up heap construction (starts iterating from end of array)
  * (attempt001) When initially building heap, heapifies every element of array 
  * ^ 9 iterations inside 'MakeHeap' func

Cluster1-

* 1x: Sift up heap construction (starts at end of array)
  * (attempt000) When initially building heap, heapifies only the leaf nodes (i.e., last half of array)
  * ^ 4 iterations inside 'MakeHeap'

Cluster2-

* 1x: Sift down heap construction (starts at beginning of array)
  * Similar to cluster 1, the difference being that cluster 2 doesn't swap elements inside the heapify function
  * ^ 6 iterations inside 'MakeHeap'
  * (attempt002) I replaced `ref` swap with regular swap b/c Pex considers these two types of swaps to be "diff strategy," when they're actually the same thing...
    * Doing this did *not* alter clustering results...they're all still in cluster by themselves

Cluster3-

* 1x: Sift up heap construction (starts at end of array)
  * (attempt004) Heapifies leaf nodes and has a recursive `heapify()` function
  * ^ 10 iterations inside 'MakeHeap'

Cluster4-

* 1x: Sift up heap construction (starts at end of array)
  * (attempt003) Also only heapifies leaf nodes in initial heap construction. Apart from this I'm unsure how it's diff from cluster 1... 
  * ^ 10 iterations inside 'MakeHeap'

Cluster5-

* 1x: Sift down heap construction (starts at beginning of array)
  * Heapifies leaf nodes and has a recursive `heapify()` function (attempt005). Apart from the # iterations, I'm not sure how it's different from cluster 3's...
  * ^ 5 iterations inside 'MakeHeap'

**Insertion Sort**

Cluster0-

All implementations in cluster 0 have two pointers: one in outer-loop that moves left to right and another in inner-loop that moves right to left. The one implementation in cluster 1 has two pointers that *both* move from left to right. 

* 5x: Cascades smallest element down to left-most part of array
  * Ex: [7,7,6,0]-> [7,**6**,7,0]-> [**6**,7,7,0]-> [6,7,**0**,7]-> [6,**0**,7,7]-> [**0**,6,7,7]

Cluster1-

* 1x: Finds the smallest element at to the right and directly swaps with it
  * Ex: [7,7,6,0]-> [**0**,7,6,**7**]-> [0,**6**,**7**,7]

**Merge Sort**

Cluster0-

* 3x: Stores left half of input array into one integer array and the right half into another integer array 
* 1x: Doesn't store the halves into extra arrays, instead uses indices to partition the arrays (attempt005)

Cluster1-

* 1x: Doesn't store the halves into extra arrays, instead uses indices to partition the arrays (attempt003)
  * ^ May be **false positive**...attempt003 and attempt005 look nearly identical...unsure why they weren't clustered together. The PCs are wildly long and unhelpful to understand why they weren't clustered together.

Cluster2-

* 1x: Stores left half of input array into a *list* and the right half into another *list*. Adds/deletes elements from lists instead of incrementing/decrementing pointers to array elements



**Quick Sort**

Cluster0-

* 2x: Pivot is right-most index

Cluster1-

* 1x: Pivot is middle index

Cluster2-

* 1x: Pivot is $$log10(array.Length)/2$$

Cluster3-

* Pivot is left-most index

Cluster4-

* Pivot is randomly chosen index



### Pex4Fun

**Puzzle 33**

Cluster0-

* 4x: Converts input int to string and returns length of string (`return x.ToString().Length`)
* 8x: Uses while-loop, dividing input int by 10 and incrementing count by 1 until the division is < 1

Cluster1-

* Returns log_10 of the input int + 1

**Puzzle 34**

Cluster0-

* 9x: Uses for-loop to print all numbers up to and including n
* 1x: Uses for-loop to print square of all numbers up to and including n, i.e., "0 1 4 9 16 ..."
  * ^ Likely **false positive** b/c we assumed that last submission is correct...but we see this isn't the case here...this doesn't yield expected answer

**Puzzle 35**

Cluster0-

* 7x: Uses for-loop to create a list of multiples of input integer n from [0, n)

**Puzzle 36**

Cluster0-

* 1x: Uses two separate for-loops to return all numbers from [0, n) three times
* 1x: Uses one for-loop to store all numbers from [0, n) into a string, then returns `string + string + string` (concatenated to itself twice)

Cluster1-

* 2x: Uses one for-loop and iterates until 3*input, then uses modulo operator to store all numbers from  [0, n) into a string, then returns result

**Puzzle 38**

Cluster0-

* 7x: Use for-loop to print string n times

**Puzzle 40**

Cluster0-

* 2x: Uses Linq expression to count all occurrences of letter 'a' in string
* 4x: Uses for-loop to count all occurrences of letter 'a' in string

**Puzzle 42**

Cluster0-

* 4x: Uses for-loop to output [0, 1, 4, ...n^2] (in ascending order), appends to result string
* 1x: Uses Linq expression to output [0, 1, 4, ...n^2]

Cluster1-

* Uses  for-loop to output [0, 1, 4, ...n^2], but starts iterating from end of array and *prepends* to result string

**Puzzle 43**

Cluster0-

* 4x: Uses for-loop to output [n^2...4, 1, 0] (descending order); starting iteration from end of array and prepends to result string
* 1x: Uses Linq expression

**Puzzle 45**

Cluster0-

* 2x: Uses for-loop to calculate factorial
* 2x: Uses recursion

**Puzzle 47**

Cluster0-

* 2x: Uses for-loop, incrementing the sum only on the even-numbered indices

**Puzzle 48**

Cluster0-

* 1x: Uses one for-loop to multiple each number from [n, 1] by its (index+1) , takes O(n)
* 1x: Uses nested for-loop to do the same work, takes O(n^2) though

**Puzzle 49**

Cluster0-

* 6x: Uses for-loop to put one "_" for length of input string

**Puzzle 50**

Cluster0-

* 6x: Uses for-loop to implement Caesar's cipher

**Puzzle 53**

Cluster0-

* 5x: Returns true if input integer less than 50, else false

**Puzzle 55**

Cluster0-

* 7x: Uses if-else statements to return the proper strings

**Puzzle 57**

Cluster0-

* 5x: Uses if-else statements to return "one" if the input integer is less than 10, "two" if it's less than 100, and "three" if it's less than 1000

Cluster1- 

* 1x: Converts input integer to string and uses if-else statements to return "one", "two", or "three" based on the length of the string

**Puzzle 58**

Cluster0-

* 6x: Uses if-else statements to return "even" if input integer is even, else returns "odd"

**Puzzle 59**

Cluster0-

* 6x: Uses if-else statements to return proper strings

**Puzzle 60**

Cluster0-

* 6x: Uses if-else statements to return proper strings

**Puzzle 61**

Cluster0-

* 4x: Uses if-else statements to return proper strings if i == 2j and j == 2k

Cluster1-

* 1x: Uses if-else statements to return proper strings by first checking if `i ` and `j` are even numbers, if they are not, return "no", else return "yes" if i/2 == j and j/2 == k, else return "no"

**Puzzle 62**

Cluster0-

* 4x: Uses if-else statements to return proper strings

**Puzzle 63**

Cluster0-

* 5x: Uses if-else statements to return proper strings

**Puzzle 64**

Cluster0-

* 1x: Continually divides input integer by 10 to check each digit to see if the original input represents a "fancy year" 

Cluster1-

* 2x: Hard-codes to test if the input is "1111" or "2222" or ... "9999"

Clsuter2-

* 1x: Stores each digit of the number into a variable via combination of division and modulus, then checks if each digit is the same

Cluster3-

* 1x: If input integer is less than 10,000 and input%1111 == 0, then it represents a fancy year

**Puzzle 65**

Cluster0- 

* 3x: Returns true if either combination (i.e., a^2 + b^2 == c^2, a^2 + c^2 == b^2, b^2 + c^2 == a^2) is true, else false
* 1x: Places integers a, b, c into an array and sorts it, then returns true if arr[0]^2 + arr[1]^2 == arr[2]^2, else false
  * ^ \*\* May be **false positive** b/c this seems like completely diff strategy than the other three...

**Puzzle 73**

Cluster0-

* 1x: Uses if-else statements to return proper boolean

**Puzzle 74**

Cluster0-

* 1x: Unconditionally returns "No"
  * ^ Must be **false positive** b/c doesn't produce right answer when input string does contain all lowercase, b/c has no condition check
* 4x: Uses if-else statements to return proper string

**Puzzle 75**

Cluster0-

* 3x: Uses for-loop to do alternate upper and lower case characters

**Puzzle 105**

Cluster0-

* 2x: Uses Linq expression `Contains()` to see if input integer is within input integer array
* 1x: Uses for-loop to see if input integer is within input integer array

 **Puzzle 107**

Cluster0-

* 2x: Uses for-loop to create integer array from [0, n], where n in input integer

**Puzzle 110**

Cluster0-

* 1x: Uses single for-loop to create integer array from [n, 0] (reverse order)
* 1x: Uses two for-loops; first one creates integer array from [0, n], then another for-loop to reverse the array in-place
  * ^ May be **false positive**...why should these be considered the same strategy if the second submission clearly did more work and used diff approach?

**Puzzle 111**

Cluster0-

* 2x: Uses for-loop to negate each element of array

**Puzzle 112**

Clsuter0-

* 2x: Use `Array.Reverse()` 
* 1x: Use for-loop to reverse the input integer array

**Puzzle 132**

Cluster0-

* 5x: Uses for-loop to count occurrence of input integer in input integer array
* 1x: Uses Linq expression (`List.FindAll()`) to count occurrence of input integer in input integer array

**Puzzle 135**

Cluster0-

* 1x: Uses for-loop to find last index of integer in input integer array; starts iterating from end of array
* 2x: Uses Linq expression (`Array.LastIndexOf()`) to find last index of integer in input integer array

Cluster1-

* 1x:  Uses for-loop to find last index of integer in input integer array; starts iterating from *beginning* of array

**Puzzle 140**

Cluster0-

* 3x: Uses for-loop to determine if array in ascending order; return true if it is, else false

**Puzzle 144**

Cluster0-

* 3x: Uses for-loop to implement Caesar's cipher with offset of seven

**Puzzle 152**

Cluster0-

* 1x: Uses one for-loop and two pointers (head, tail) to see if characters at both pointers are same letter (i.e., is the input string a palindrome)
* 1x: Stores both halves of the input string into two variables, reverses one, and returns true if they're equal, else false



### Peking University 

**Homework 1**

For all submissions in cluster 0, they have an outer-loop which iterates `j` from 11 to the input integer. Then there's an inner-loop which checks to see if `j` is prime. Cluster 1 *doesn't* use such an inner-loop.

Cluster0-

* 22x: Checks to see if all positive integers from [2, 3, 4, ...n] are factors of the integer n in question
* 6x: Checks if *odd* positive integers from [3, 5, ...sqrt(n)] are factors of the integer n in question
* 20x: Checks if all positive integers from [2, 3, 4, ...sqrt(n)] are factors of the integer n in question
* 10x: Checks if all positive integers from [2, 3, 4, ...(n/2)] are factors of the integer n in question
* 4x: Checks if all *odd* positive integers from [3, 5, ...(n/2)] are factors of the integer n in question

Cluster1-

* 1x: First makes list of prime integers, then increments the running sum for each prime integer with the same beginning and ending digit values
  * Checks if all positive integers from [2, 3, 4, ...sqrt(n)] are factors of the integer n in question
  * Doesn't use inner-loop to check if integer in question in prime

**Homework 2**

For some reason, Pex reports "limitation in floating point conversion" when comparing a double to an int (e.g., `if (myDouble >= myInt)...`). This may affect clustering.

Cluster0-

* 21x: Stops computing summation once running sum >= input integer
  * Has PC: `input > 0 && input <= 1` and returns `1` for input integer 1
* 183x: Stops iterating once running sum > input integer
  * **To verify that the following is true for all 183 subs:** returns `2` for input integer 1 and has PC: `input >= 1 && input < 1.5` b/c of `if (sum > input)...` inside for-loop

Cluster1-

* 5x: Stops computing summation once running sum >= input integer
  * Has PC: `input <= 1`
* 1x: Stops iterating once running sum > input integer
  * Outputs `2` for input integer 1 b/c of do-while structure
  * Has PC: `input < 1.5`
* ^ *All six* in cluster 1 likely **false positives** b/c they all use exactly same strategy. PCs fail here b/c placement of conditional statements differ based on loop type used (i.e., for, while, do-while). 

**Homework 3**

The difference here is that the submissions in cluster 1 define a lower bound on the range of values for each array element, whereas those in cluster 0 do not.

Cluster0-

* 517x: Have if-statements for the following cases: 
  * (1) element < 100
  * (2) element in range [100, 200)
  * (3) element in range [200, 500)
  * (4) element >= 500

Cluster1-

* 9x: Have if-statements for the following cases:
  * (1) *element in range [0, 100)* 
  * (2) element in range [100, 200)
  * (3) element in range [200, 500)
  * (4) element >= 500







****



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

Skipping CodeHunt's <u>Sector3-Level5</u> and Pex4Fun's puzzle numbers <u>123</u>, <u>124</u>,  because the puzzles return `int[][]`, which Pex cannot implicitly cast to a Path Condition string. I'm sure a clever workaround could be fashioned, but I will not spend time on that at this time.

April 22nd, 2020:

Skipping PKU's hw4 because it was taking *way* to long to cluster. This puzzle only 13 students being evaluated, but `cluster.py` ran on the order of several hours...so I'll skip this one as well. 

April 25th, 2020:

We're also skipping Pex4Fun's puzzle 37, 39, 41, 44, 46, 56, 106, and 109, 133. For some of them, it's because our tool failed to produce clusters for...I can't remember precisely why we ignore others, but it likely has to do with our tool not being able to execute `runseed.py` or `cluster.py` on them. I'll look back into it later.

May 6th, 2020:

Skipping CodeHunt's Sector3-Level3 because the student submissions and instructor solution don't have > 1 branch. I somehow missed this before, so now I'm ignoring it.

May 8th, 2020:

Skipping Pex4Fun's Puzzle 83, 141, and 143 because our tool failed to cluster their submissions, for some reason. Ignoring them for now.



## Bugs Fixed

March 18th, 2020:

Found a bug where preambles with `&&` operators were getting split up, which affected our clustering results. Here's an example of a preamble that would cause issues:

PC1 = ` bool s0 = a[0] > a[1]; bool s1 = s0 && a[2] > a[1]; return 5`

The above expression for`s1` would get split into two terms, which in many cases is undesirable. Fixed this bug by not splitting up preamble expressions.