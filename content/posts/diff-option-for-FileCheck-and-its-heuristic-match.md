---
title: "Diff Option for LLVM FileCheck"
date: 2026-04-01T14:36:59+05:30
draft: false
---

### Introduction
FileCheck is an LLVM pattern matching tool which is used to validate the compiler output. It compares "Check Prefixes" (expected patterns) against the "Input" (the actual compiler output). FileCheck is used in many projects like LLVM, MLIR, LLD, Clang, Swift etc.   


### Problem
If you have worked with LLVM, you must have written FileCheck based test cases too. It is a powerful tool, but when a complex test case fails, the output can become a daunting task to grasp. You often get a "Match not found" error, followed by a dump of the input text with FileCheck trying to match with right line. In practice, this can take a significant amount of time for complex test cases.


### Before and After view
To illustrate the impact, I modified the expected assembly in two places.

![Old FileCheck](/images/old-filecheck.png)

And after the --diff=split option
![New FileCheck](/images/new-filecheck.png)

We now can see the same or more information in 1/5 part (80 vs 15 lines) of the data from the previous standard view with proper colouring. With side-by-side positioning and color-coding, the mismatch becomes immediately obvious. 

There are both split view and unified view for users to choose from. It also supports variable substitution which means you can see the expected substituted value or raw regex pattern.  


### Implementation 
The implementation required a major shift in how FileCheck handles diagnostics. Previously, FileCheck was "fail-fast", with --diff option it behaves as a diff engine,
by introducing the statefulness to the error reporting.

Terminology -  
CheckRegion = The "pool" of compiler output currently being searched.  
CheckStr = The "rule" we are trying to satisfy, CHECK lines.  
ActualLine = The corresponding line from input(compiler output)  
ExpectedLine = One line of CheckStr - CHECK prefix.  
TargetLine = The "best guess" of what the compiler output actually was instead of what we wanted.  


#### Mismatch Tracker
CheckInput function is the entry point for our diff outout. It process each CHECK line of each CHECK-LABEL. If any of the CHECK mismatched it breaks
the loop for that CheckRegion and report the error before continuing for next CheckRegion, marked by CHECK-LABEL. CHECK-LABEL is act as resyncronisation point.

We have intercepted here and call out handleDiffFailure function which will handle the mismatch failures. handleDiffFailure do the reverse search in Diag vector which
contains the fuzzy mismatches log. If this founds a match in diag vector we call it TargetLine/ActualLine/Input Line and if nothing matches in diag vector, our TargetLine
assume to be next line of input. Once we have the target line, we call printDiff with TargetLine and CheckStr(contain CHECK lines) to print the mismatches.

In printDiff we get the expected line(CHECK prefixed) from CheckStr and we have TargetLine already. Next we gather the Diff Context and call renderDiff with ExpectedLineNo,
ActualLineNo, ExpectedLine, ActualLine, Context. Once mismatches print we advance the CheckRegion(Input Region) to match with next CHECK line.

renderDiff simply render the diff, both split and unified view to standard ouput with proper colouring as used in diff tools.  

#### Diff Context Window
Printing only the mismatches are not enough to grasp the developer where the mismatches happening, providing them a context of one line above and below will help to spot the mismatche region. getDiffContext uses a struct to keep track of ActualLine, a line below it and a line above it. 

#### Variable Substitution
FileCheck have variables that can be define to value, and further these variable can be use. Controlling the variable subsitution help developer to find what FileCheck was 
expecting with regard to variables.
For substituting the variables, I have used the FileCheck pre-calculated Substitutions vector which is prefilled with variable and their value.  function takes the expectedLine find the `[[`  to get variable which we want to subsitute. From Substitutions vector we get substitution and clean them to get CleanValue which will replace the variable that we got from expectedLine. So in that each variable from left to right in ExpectedLine is subsituted with substitution vector elements.

#### Creating the Split-View Layout
Split is also popular choice of diff viewing. For that just updated the renderDiff to print mismatches with alignment. It has one nice feature that mismatchLine is printed
with getCenteredView function which make the long line mismatches centered on mismatched position by truncating the left and right part of Expected and Actual string and filling with `...`. This ensures that even long lines remain readable by focusing on the mismatched region.

#### Handling CHECK-NOT Logic 
Handling for CHECK-NOT was different since it is use for prohibiting the matches. We intercept matches in printMatch, get the match pos and route violations through the diff pipeline printDiff function.

### Pull requests
This feature is currently in review and is expected to land in the coming weeks.  
The work was split into several logical steps:

[FileCheck] Add a diff output option for FileCheck  
https://github.com/llvm/llvm-project/pull/187120

[FileCheck] Add split view diff option for FileCheck  
https://github.com/llvm/llvm-project/pull/189269

[FileCheck] Add variable substitution for diff option  
https://github.com/llvm/llvm-project/pull/189273

[FileCheck] Add check-not support for diff option  
https://github.com/llvm/llvm-project/pull/189357

[FileCheck] Add check-label support for diff option  
https://github.com/llvm/llvm-project/pull/189359


### Final words

I believe this is a significant improvement for developers productivity, they will spend less time interpreting the test case failures. It helps new comers to understand what their test case failure mean, as I have seen my newcomer friend getting overwhelmed by seeing the FileCheck output. 

I would like to thank Henrik G. Olsson for his patience and thorough reviews of such a large change. With these PRs most work is completed. but I do find out that many places FileCheck present wrong things since it is only do greedy heuristic search for mismatches.  


```  
$ cat input.txt                                                                                                                             
ALPHA
GAMMA
```

``` 
$ cat check.txt                                                                                                                                       
; CHECK: ALPHA
; CHECK: BETA
; CHECK: GAMMA
```

```
$ bin/FileCheck ./check.txt --input-file=./input.txt --diff=split 
--- ./check.txt
+++ ./input.txt
@@ -2 +2 @@
 ALPHA
-BETA
+GAMMA

FileCheck: Found 1 unique textual mismatch.
```

Here FileCheck tried to match BETA and GAMMA, and did not care to check further that GAMMA is also present in next CHECK line. That can be improve by doing small look ahead search for the matches but that is reserved for some other day. 