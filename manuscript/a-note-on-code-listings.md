# A Note on Code Listings
The code formatting in this book only allows for about 60-odd characters per line. We've tried our best to keep our code within that limit, although sometimes you may see some awkward formatting as a result.

For example:

```PowerShell
       Invoke-CimMethod -ComputerName $computer `                        -MethodName Change `                        -Query "SELECT * FROM Win32_Service WHERE Name = '$ServiceName'" `
```

Here, you can see the default action for a too-long line - it gets word-wrapped, and a backslash inserted at the wrap point to let you know. We try to avoid those situations, but they may sometimes be unavoidable. When we _do_ avoid them, it may be with awkward formatting, such as in the above where we used backticks (`) or:

```PowerShell
       Invoke-CimMethod -ComputerName $computer `                        -MethodName Change `         -Query "SELECT * FROM Win32_Service WHERE Name = '$ServiceName'" `
```

Here, we've given up on neatly aligning everything to prevent a wrap situation. Ugly, but oh well.

You may also see this crop up in `inline code` snippets, especially the backslash. 


I> If you are reading this book on a Kindle, tablet or other e-reader, then we hope you'll understand that all code formatting bets are off the table. There's no telling what the formatting will look like due to how each reader might format the page. We trust you know enough about PowerShell to not get distracted by odd line breaks or whatever.

When _you_ write PowerShell code, you should not be limited by these constraints. There is no reason for you to have to use a backtick to "break" a command. Simply type out your command. If you want to break a long line to make it easier to read without a lot of horizontal scrolling, you can hit `Enter` after any of these characters:

* Open parenthesis (
* Open curly brace {
* Pipe |
* Comma ,
* Semicolon ;
* Equal sign =

This is probably not a complete list, but breaking after any of these characters makes the most sense.

Anyway, we apologize for these artifacts. 

## Code Samples
Longer code listings are available at http://github.com/concentrateddon/TheDSCBookCode. Shorter listings - because they're often not useful as-is - are not necessarily replicated there. We really mean for most of the code in this book to be _illustrations,_ not copy-and-paste usable.
