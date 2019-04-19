# <center>CodeSimilarity v. 2</center>

*April 14, 2019*





## Overview

- [CodeHunt Sector 2 Level 3](#Puzzle Description (Sector 2 Level 3))
  - [Solution](#Solution)
  - [PUT](#PUT)
  - [Manual Inspection](#Manual Inspection)
    - [User014](#User014 (3 correct submissions))
      - [Submission 1 (2/3)](#Submission 1 (2/3))
      - [Submission 2 (2/3)](#Submission 2 (2/3))
      - [Submission 3 (3/3)](#Submission 3 (3/3))



## Content

### Puzzle Description (Sector 2 Level 3)

```
P258-1: Reverse all but first & last characters of a string
Used in Imagine Cup 2014```
```

### Solution

```c#
using System;
using Microsoft.Pex.Framework;

public class Program {
  public static string Puzzle(string s) {
    PexAssume.IsNotNull(s);
    PexAssume.IsTrue(s.Length >= 3);
    int len = s.Length;
    if (s == "codehunt");
	if (s == "abcabc");
    for(int i=0; i<len; i++)
	    PexAssume.IsTrue(s[i] == ' ' | (s[i]>='a' & s[i]<='z'));
    string c = "";
    c += s[0];
    for (int i = len-2; i >= 1; i--) {
      c += s[i];
    }
    c += s[len-1];
    return c;
  }
}
```

### PUT

TODO: This PUT will generate input like "aaaa" which cannot identify whether the submission code reversed the string or not.

```c#
using System;
using Microsoft.Pex.Framework;
using Microsoft.Pex.Framework.Validation;
using Microsoft.VisualStudio.TestTools.UnitTesting;

/// <summary>This class contains parameterized unit tests for Program</summary>
[PexClass(typeof(global::Program))]
[PexAllowedExceptionFromTypeUnderTest(typeof(InvalidOperationException))]
[PexAllowedExceptionFromTypeUnderTest(typeof(ArgumentException), AcceptExceptionSubtypes = true)]
[TestClass]
public partial class ProgramTest
{
    /// <summary>Test stub for Puzzle(Int32[])</summary>
    [PexMethod]
    public string Puzzle(string a)
    {
        PexAssume.IsTrue(a != null);
        PexAssume.IsTrue(a.Length >= 3);
        for(int i = 0; i < a.Length; i++)
	    	PexAssume.IsTrue(a[i] == ' ' | (a[i]>='a' & a[i]<='z'));
        string result = global::Program.Puzzle(a);
        return "PC: " + PexSymbolicValue.GetPathConditionString() + " RET: " + PexSymbolicValue.ToString<string>(result);;
        // TODO: add assertions to method ProgramTest.Puzzle(Int32[])
    }
}
```

### Manual Inspection

#### User014 (3 correct submissions)

##### Submission 1 (2/3)

Student's code:

```c#
using System;

public class Program {
  public static string Puzzle(string s) {
    if(s.Length<=3)return s;
	int i = 1, j = s.Length-2;
	char[] chrs = s.ToCharArray();
	while(i<j){
		char t = chrs[j];
		chrs[j] = chrs[i];
		chrs[i] = t; 
		++i;--j;
	}
	return new String(chrs);
  }
}
```

Test cases:

```c#
public partial class ProgramTest
{
    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle18()
    {
        string s;
        s = this.Puzzle("aaaaaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && \r\n       (a[3] == \' \' || 96 < (int)((ushort)(a[3])) && (int)((ushort)(a[3])) < 123) && \r\n       (a[4] == \' \' || 96 < (int)((ushort)(a[4])) && (int)((ushort)(a[4])) < 123) && \r\n       (a[5] == \' \' || 96 < (int)((ushort)(a[5])) && (int)((ushort)(a[5])) < 123) && a.Length == 6; RET: string s0 = new;\r\nreturn s0;", 
             s);
    }

    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle295()
    {
        string s;
        s = this.Puzzle("aaaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && \r\n       (a[3] == \' \' || 96 < (int)((ushort)(a[3])) && (int)((ushort)(a[3])) < 123) && a.Length == 4; RET: string s0 = new;\r\nreturn s0;", 
             s);
    }

    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle863()
    {
        string s;
        s = this.Puzzle("aaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && a.Length == 3; RET: return a;", 
             s);
    }
}

```

##### Submission 2 (2/3)

Student's code:

```c#
using System;
using System.Text;
public class Program {
  public static string Puzzle(string s) {
    if(s.Length<=3)return s;
	int i = 1, j = s.Length-2;
	StringBuilder sb = new StringBuilder(s);
	//char[] chrs = s.ToCharArray();
	while(i<j){
		char t = sb[j];
		sb[j] = sb[i];
		sb[i] = t; 
		++i;--j;
	}
	return sb.ToString();
  }
}
```

Test cases:

```c#
public partial class ProgramTest
{
    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle18()
    {
        string s;
        s = this.Puzzle("aaaaaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && \r\n       (a[3] == \' \' || 96 < (int)((ushort)(a[3])) && (int)((ushort)(a[3])) < 123) && \r\n       (a[4] == \' \' || 96 < (int)((ushort)(a[4])) && (int)((ushort)(a[4])) < 123) && \r\n       (a[5] == \' \' || 96 < (int)((ushort)(a[5])) && (int)((ushort)(a[5])) < 123) && a.Length == 6; RET: return \"aaaaaa\";", 
             s);
    }

    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle295()
    {
        string s;
        s = this.Puzzle("aaaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && \r\n       (a[3] == \' \' || 96 < (int)((ushort)(a[3])) && (int)((ushort)(a[3])) < 123) && a.Length == 4; RET: return \"aaaa\";", 
             s);
    }

    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle863()
    {
        string s;
        s = this.Puzzle("aaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && a.Length == 3; RET: return a;", 
             s);
    }
}
```

##### Submission 3 (3/3)

Student code:

```c#
using System;
using System.Text;
public class Program {
  public static string Puzzle(string s) {
    //if(s.Length<=3)return s;
	int i = 1, j = s.Length-2;
	StringBuilder sb = new StringBuilder(s);
	//char[] chrs = s.ToCharArray();
	while(i<j){
		char t = sb[j];
		sb[j] = sb[i];
		sb[i] = t; 
		++i;--j;
	}
	return sb.ToString();
  }
}
```

Test cases:

```c#
public partial class ProgramTest
{
    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle18()
    {
        string s;
        s = this.Puzzle("aaaaaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && \r\n       (a[3] == \' \' || 96 < (int)((ushort)(a[3])) && (int)((ushort)(a[3])) < 123) && \r\n       (a[4] == \' \' || 96 < (int)((ushort)(a[4])) && (int)((ushort)(a[4])) < 123) && \r\n       (a[5] == \' \' || 96 < (int)((ushort)(a[5])) && (int)((ushort)(a[5])) < 123) && a.Length == 6; RET: return \"aaaaaa\";", 
             s);
    }

    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle295()
    {
        string s;
        s = this.Puzzle("aaaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && \r\n       (a[3] == \' \' || 96 < (int)((ushort)(a[3])) && (int)((ushort)(a[3])) < 123) && a.Length == 4; RET: return \"aaaa\";", 
             s);
    }

    [TestMethod]
    [PexGeneratedBy(typeof(global::ProgramTest))]
    public void Puzzle863()
    {
        string s;
        s = this.Puzzle("aaa");
        Assert.AreEqual<string>
            ("PC: return a != (string)null && \r\n       (a[0] == \' \' || 96 < (int)((ushort)(a[0])) && (int)((ushort)(a[0])) < 123) && \r\n       (a[1] == \' \' || 96 < (int)((ushort)(a[1])) && (int)((ushort)(a[1])) < 123) && \r\n       (a[2] == \' \' || 96 < (int)((ushort)(a[2])) && (int)((ushort)(a[2])) < 123) && a.Length == 3; RET: return \"aaa\";", 
             s);
    }
}
```



