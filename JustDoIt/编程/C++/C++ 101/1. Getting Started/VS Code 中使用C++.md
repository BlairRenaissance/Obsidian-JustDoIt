### 创建新cpp文件

To create a new file, choose _View > Explorer_ from the top nav to open the Explorer pane, and then click the _New File_ icon to the right of the project name. Alternately, choose _File > New File_ from the top nav. Then give your new file a name (don’t forget the .cpp extension). If the file appears inside the _.vscode_ folder, drag it up one level to the project folder.

Next open the _tasks.json_ file, and find the line `"${file}",`.

You have two options here:

- If you wish to be explicit about what files get compiled, replace `"${file}",` with the name of each file you wish to compile, one per line, like this:

`"main.cpp",`  
`"add.cpp",`

- Reader “geo” reports that you can have VS Code automatically compile all .cpp files in the directory by replacing `"${file}",` with `"${fileDirname}\\**.cpp"` (on Windows).
- Reader “Orb” reports that `"${fileDirname}/**.cpp"` works on Unix.

---

### 包含其它路径下的.h文件

一个（不好的）方法是包含头文件的相对路径：
```
#include "headers/myHeader.h"
#include "../moreHeaders/myOtherHeader.h"
```

但显然一变架构代码就不可用了。

更好的方法是告诉你的编译器或 IDE，你在其他位置有一堆头文件。

For VS Code users

In your _tasks.json_ configuration file, add a new line in the _“Args”_ section:  
`"-I./source/includes",`

There is no space after the `-I`. For a full path (rather than a relative path), remove the `.` after `-I`.

---

### 进行 Debug

To set up debugging, press _Ctrl+Shift+P_ and select “C/C++: Add Debug Configuration”, followed by “C/C++: g++ build and debug active file”. This should create and open the `launch.json` configuration file. Change the “stopAtEntry” to true:  
`"stopAtEntry": true,`

Then open _main.cpp_ and start debugging by pressing _F5_ or by pressing _Ctrl+Shift+P_ and selecting “Debug: Start Debugging and Stop on Entry”.

In VS Code : 
- **step into** : the _step into_ command can be accessed via _Run > Step Into_.
- **step over** : the _step over_ command can be accessed via _Run > Step Over_, or by pressing the _F10_ shortcut key.
- **step out** : the _step out_ command can be accessed via _Run > Step Out_, or by pressing the _shift+F11_ shortcut combo.
- **run to cursor** : the _run to cursor_ command can be accessed while already debugging a program by right clicking a statement in your code and choosing _Run to Cursor_ from the context menu.
- **continue** : the _continue_ command can be accessed while already debugging a program via _Run menu > Continue_, or by pressing the _F5_ shortcut key.
- **start** : the _start_ command can be accessed while not debugging a program via _Run menu > Start Debugging_, or by pressing the _F5_ shortcut key.
- **set or remove breakpoint** : you can set or remove a breakpoint via _Run menu > Toggle Breakpoint_, or by pressing the _F9_ shortcut key, or by clicking to the left of the line number.
- 