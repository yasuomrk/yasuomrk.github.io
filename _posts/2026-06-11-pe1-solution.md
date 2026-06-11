---
layout: post
title: "Project Euler Problem #1 Solution"
date: 2026-06-11
category: maths
---

## overview
The problem is actually very easy. The description is:
> If we list all the natural numbers below 10 that are multiples of 3 or 5 , we get 3, 5, 6 and 9. The sum of these multiples is 23. Find the sum of all the multiples of 3 or 5 below 1000.

## solution
For this problem, we can use summation. let’s start by writing the summation formula:
<img src="/assets/images/maths/post_1/latex_1.png" style="display:block; margin:1.2rem auto;">
We need to find the sum of all numbers divisible by 3 up until 999, because 999 is the highest number below 1000 that is perfectly divided by 3. To do this, we substitute a=3 and d=3, where a is the starting term and d is the common difference between the terms:
<img src="/assets/images/maths/post_1/latex_2.png" style="display:block; margin:1.2rem auto; width:40%;">
*(N is the highest number below 1000 that is divided by a).*
Now, the same formula applies to sum of all numbers below 1000, divided by 5:
<img src="/assets/images/maths/post_1/latex_3.png" style="display:block; margin:1.2rem auto; width:44%;">
Now, we add S_n1 and S_n2:
<img src="/assets/images/maths/post_1/latex_4.png" style="display:block; margin:1.2rem auto;">
And we are done, right…? No, because in there we have numbers that are both multiples of 3 AND 5, twice. For example:
```
3, 6, 9, 12, 15, 18...
5, 10, 15, 20...
```
See above how 15 is in there twice? To fix this issue, what we can do is find the least common multiple (LCM) of 3 and 5, and we apply whatever formula we applied to these two previously. Then, the sum of all numbers below 1000, divisible by 15, will be subtracted from S_n1,2 and we get the answer:
<img src="/assets/images/maths/post_1/latex_5.png" style="display:block; margin:1.2rem auto; width:40%;">
Now, we subtract:
<img src="/assets/images/maths/post_1/latex_6.png" style="display:block; margin:1.2rem auto; width:50%;">
So, the sum of all the multiples of 3 or 5 below 1000 is 233.168!
Ultimately, we can also use sigma notation as it’s no different, but I found arithmetic progressions much simpler because I’ve also done them in school. :). Also, I'm using `.` instead of `,` because I'm not American, baby.
<br>
- *LaTeX site used: [https://latexeditor.lagrida.com](https://latexeditor.lagrida.com/)*