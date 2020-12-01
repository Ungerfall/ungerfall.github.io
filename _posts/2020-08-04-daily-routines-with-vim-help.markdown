---
layout: post
title:  "Daily routines with Vim help"
---

As a software developer I occasionally encounter tasks which require some 
text processing. Lucklily, [Vim](https://www.vim.org/) provides a large 
amount of commands to help doing it with joy.

### Sorting CI/CD stages time spent
Our jenkins CI/CD pipeline shows output results in the following format:
``` bash
Task                                                           Duration            
-------------------------------------------------------------------------------
[task name]                                                    hh:MM:ss.fffffff
```
So, when total build time exceeds expectations I like to see which task slows 
down the process. Normally I would use `:sort`, however it sorts text 
lexicographically. To cope with this we need to move duration column to the 
leftmost side by using `visual block mode` and then proceed with `:sort`. 
The whole process you can see below by clicking on the expand button  
![sequence](/assets/azure-workshop-sequence.png)  
<details>
  <summary>Click to expand<summary>
  ![vim-sort](/assets/vim-sort.gif)
</details>
The `:sort` command has additional attributes, for example, you can sort in 
descending order adding `-h` argument. More details: `:help :sort`  
