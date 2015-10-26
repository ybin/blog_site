title: IntelliJ Idea快捷键
date: 2015-02-06 16:42:14
categories: 工具
tags: [java, 工具, intellij]
---

IntelliJ Idea快捷键记录。

<!-- more -->
# IntelliJ Idea shortcuts

## General

Ctrl + Tab
    Switch between tabs and tool window
    注：按下Ctrl + Tab会弹出Popup，然后使用方向键进行选择，松开Ctrl + Tab进行跳转。
        如果只是想切换Tab而非tool window，则可用Alt + Left/Right
Ctrl + Shift + A
    Find Action
    

## Search/Replace

Ctrl + F
    Find
Ctrl + Shift + F
    Find in path
    注：在整个project/module/directory里面搜索，"Replace in path"类似。
Ctrl + R
    Replace
Ctrl + Shift + R
    Replace in path
Alt + F7
    Find usages
Ctrl + F7
    Find usages in file
Ctrl + Shift + F7
    Highlight usages in file
Ctrl + Alt + F7
    Show usages


## Refactoring

F5
    Copy
    注：复制当前类到一个新类，或者作为另一个类的内部类等，有对话框可选择。
F6  Move
    注：移动当前类到一个新类。
Alt + Delete
    Safe Delete
Shift + F6
    Rename
    注：不解释！
Ctrl + F6
    Change Signature
Ctrl + Alt + N
    Inline
Ctrl + Alt + M
    Extract Method
    注：提取代码块作为一个方法
Ctrl + Alt + V
    Extract Variable
    注：提取代码块作为一个变量
Ctrl + Alt + F
    Extract Field
    注：提取代码块作为一个属性
Ctrl + Alt + C
    Extract Constant
    注：提取代码块作为一个常量
Ctrl + Alt + P
    Extract Parameter
    注：提取代码块作为一个参数


## Navigation

Ctrl + N
    Go to class
Ctrl + Shift + N
    Go to file
Ctrl + Alt + Shift + N
    Go to symbol
    注：不论是类名、文件名、方法、属性，统统能看得到。
Alt + Right/Left
    Go to next/previous editor tab
Ctrl + G
    Go to line
Ctrl + E
    Recent files popup
Ctrl + Alt + Left/Right
    Navigate back/forward
    注：前进、后退，相当于工具栏上的前进后退箭头，go to declaration之后，
        也是用这个返回。
Ctrl + Shift + Backspace
    Navigate to last edit location
Ctrl + B or Ctrl + Click
    Go to declaration
Ctrl + Alt + B
    Go to implementation(s)
Ctrl + U
    Go to super-method/super-class
    注：看父类的方法定义时很方便。
Alt + Up/Down
    Go to previous/next method
    注：在方法间跳转，浏览代码很方便。
Ctrl + 左方括号/右方括号
    Move to code block end/start
Ctrl + F12
    File structure popup
    注：Ctrl+F12，然后输入方法、属性名称便可快速定位过去，模拟Eclipse里面的
        Ctrl+O的功能。
Ctrl + H
    Type hierarchy
Ctrl + Shift + H
    Method hierarchy
Ctrl + Alt + H
    Call hierarchy
    注：这个太重要了，直接查看调用结构。

F2 / Shift + F2
    Next/previous highlighted error
    注：解决error时可以快速定位。
F4 / Ctrl + Enter
    Edit source / View source

        
## Editing

Ctrl + Space
    Basic code completion(the name of any class, method or variable)
    注：跟输入法冲突，一般用Alt+/替代，原来的Cyclic Expand Word(Alt+/)置空。
Ctrl + Shift + Space
    Smart code completion(filters the list of methods and variables by expected type)
Ctrl + P
    Parameter info(within method call arguments)
Ctrl + X
    Cut current line or selected block to clipboard
Ctrl +Ｙ
    Delete line at caret
    注：选中多行时，所有涉及到selected block的行均被删掉
    
Ctrl + Q
    Quick documentation lookup
    注：显示文档内容
Shift + F1
    External Doc
Alt + Insert
    Generate code... (Getters, Setters, Constructors, hashCode/equals, toString)
    注：快速生成代码。
Ctrl + O
    Override methods
Ctrl + I
    Implement methods
Ctrl + Alt + T
    Surround with… (if..else, try..catch, for, synchronized, etc.)
    注：快速插入代码。
Ctrl + /
    Comment/uncomment with line comment
Ctrl + Shift + /
    Comment/uncomment with block comment
Ctrl + W
    Select successively increasing code blocks
    注：循序渐进的选中，很有用。
Alt + Q
    Context info
    注：如显示当前所在的类信息。
Ctrl + Alt + L
    Reformat code
    注：可以reformat选中的代码、整个文件。
Ctrl + Alt + O
    Optimize imports
    注：重排import代码
Ctrl + Alt + I
    Auto-indent line(s)
    注：自动缩进选中的代码。
Tab / Shift + Tab
    Indent/unindent selected lines
Ctrl + X or Shift + Delete
    Cut current line or selected block to clipboard
    注：剪切，不仅是删除，Ctrl + Y才是删除。
Ctrl + D
    Duplicate current line or selected block
Ctrl + Shift + J
    Smart line join
    注：合并当前行和下一行
Alt + Enter
    Show intention actions and quick-fixes
    注：显示推荐的修改方案
Ctrl + Enter
    Smart line split
Shift + Enter
    Start new line
Ctrl + Shift + Enter
    Complete statement
    注：如自动插入分号
Ctrl + Shift + U
    Toggle case for word at caret or selected block
    注：大小写切换