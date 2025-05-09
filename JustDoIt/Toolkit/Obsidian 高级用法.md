## 提示框

Obsidian 的官方写法借助了 Markdown 中的引言块写法：`>` ，在后面加上 `[!标注类型]` 就可以添加对应种类的标注框了。

只需 command+H 设置成引言块，再在首行添加 `[!标注类型]`即可。

| 标注类型     | 别名                     |
| -------- | ---------------------- |
| Note     |                        |
| Abstract | `summary`, `tldr`      |
| Info     |                        |
| Todo     |                        |
| Tip      | `hint`, `important`    |
| Success  | `check`, `done`        |
| Question | `help`, `faq`          |
| Warning  | `caution`, `attention` |
| Failure  | `fail`, `missing`      |
| Danger   | `error`                |
| Bug      |                        |
| Example  |                        |
| Quote    | `cite`                 |
像这样：

```
> [!info]
> 这是条提示信息
```

## 表格

创建表格：

```
|title|title|
|---|---|
```

## 文档属性

在紧贴着文档标题的第一行打下 `---` 并回车，就会出现文档属性的选择栏。