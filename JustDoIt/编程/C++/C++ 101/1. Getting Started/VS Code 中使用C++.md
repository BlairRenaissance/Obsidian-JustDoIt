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

