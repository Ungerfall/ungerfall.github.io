---
layout: post
title:  "Vim as an AstroGrep editor"
---

[AstroGrep](http://astrogrep.sourceforge.net/) is a Microsoft Windows grep utility. Grep is a UNIX command-line program which searches within files for keywords. AstroGrep supports regular expressions, versatile printing options, stores most recently used paths and has a "context" feature which is very nice for looking at source code.

By default it opens file using Notepad. We can improve it a little bit with Vim and its ability to open a file with search using `gvim.exe +/regex filename`  

1. Go to options (Tools->Options->Text Editors) and Edit the Notepad.
2. Choose the GVim executable, for example C:\Program Files\Vim\vim90\gvim.exe
3. Edit Command Line: `+/"%4" %1`, where %1 - File, %4 - Searched Text
4. Click Ok

Once you configure it the files will opened in Vim and the searched text will be highlighted.

### See also
`:help starting.txt`

