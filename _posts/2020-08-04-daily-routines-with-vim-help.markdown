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
down the process. Normally I would use `:sort` 
