title: 清理Android APP无用资源
date: 2014-12-19 08:44:03
categories: 工具
tags: [android, python, regex]
---

用Python 3实现的一个基于android lint的Android app resource清理工具。

<!-- more -->

Android app resource清理工具已经有很多了，接触过的几个有些是纯bash实现的，
太慢，有些是只清理无用的图片资源，功能单一，于是自己用python重新实现一个，
代码很简单，托管在github上：[clean unused android res][code_repo]。

它是基于android lint实现的，即先用lint生成无用资源清单，然后我们的工具会
根据这个清单来处理这些无用资源：png, raw, menu, anim, layout等无用资源会
被删除，而string, array, string-array, color, dimen等无用资源会把无用的
xml节点删除掉而不是整个xml文件。

另外，清理无用的xml节点一个很自然的方式是用python的xml库，但是ElementTree
默认会把注释部分删除掉，需要很geek的办法才能避免，而像lxml等库又过于庞大，
所以这部分是用正则表达式来实现的，对，一个复杂的正则表达式来匹配需要删除的
节点。

### 使用方法

用法很简单：`clean_unused_android_res.py --help`有详细的用法。
你可以让它自己调用lint生成log然后再根据log进行资源清理，这需要你的lint位于
系统PATH中，也可以自己生成lint log然后用`-f`或`--file`选项告诉它。
如有必要，可以多次运行该工具，防止lint得到的无用资源不够全面。

### 实现原理

原理也很简单，根据lint log生成一个字典，

```python
{
    removable : [ removable file list ],
    xml       : {
        xml_file_name_0 : [ unused node name list ],
        xml_file_name_1 : [ unused node name list ],
        xml_file_name_2 : [ unused node name list ],
        # ......
    }
    # 如有其他类型资源还可以自己添加
}
```

removable的文件直接删除，xml类型的文件则只删除相关节点。注意有些xml文件，
如layout, anim，他们是可以直接删除的，所以归为removable类型而不是xml类型，
xml类型特指向string, array, color, dimen等不能整个文件删除的资源。

删除xml节点时，我们使用正则表达式实现。生成的字典中包含每个xml文件中需要删除
的节点的`name`属性的值，我们就是根据这些值来定位节点，从而删除节点，正则表达式
如下，

```python
# name: 每个xml文件中需要删除的xml节点的name属性的值
# cstr: xml文件的内容，对，整个文件的内容
# l   : 每个需要处理的xml文件都有一个对应的列表，保存被删除节点的name属性值

def remove_xml_nodes_p(xml, l):
    """
    xml: xml file path
    l  : value of name attri of node to be removed
    """
    if not os.path.exists(xml):
        return
    f = open(xml, 'r', encoding='utf-8')
    cstr = f.read()
    f.close()
    for name in l:
        _p = r"""
            .*\n                        # node之前的内容，我们不关心它
            (                           # 开始匹配node的内容
            \s*<\s*([a-zA-Z-]+)         # tag的开始部分，如' < string '，注意里面运行空白字符
            [^<>]*?                     # 其他可能存在的属性
            name\s*=\s*"                # name属性，如'name = "pref_camera_xxx"'
            """ + name + r"""           # name属性值
            "[^<>]*?>                   # tag开始部分的结束'>'符号，以及其他可能存在的属性
            .*?                         # node中除去tag之外的具体内容，ungreedy模式，否则会导致异乎寻常的大量匹配
            </\s*\2\s*>                 # tag的结束部分，\2引用了开始部分的tag内容
            [ \t]*\n*                   # tag结束之后可能还有空白，直到换行符的空格、制表符都删除掉
            )                           # 结束匹配node的内容
            .*                          # node之后的内容，我们同样不关心它
            """
        m = re.compile(_p, re.S | re.X).match(cstr)
        if m:
            node = m.group(1)
            node_index = cstr.find(node)
            cstr = str(cstr[:node_index] + cstr[node_index+len(node):])
            # print('remove xml node_pp: ', node)
    f = open(xml, 'w', encoding='utf-8')
    f.write(cstr)
    f.close()
```

更形象一点，图中有些表述不是很准确(生成这个图的工具是js专用的)，如开启`re.S`
模式后，`.`可以匹配换行符。建议下载大图查看。

![匹配节点的正则表达式](/res/img/android_xml_node_regex.png)

注意清理被删除节点与上一个节点、下一个节点之间的空白和换行，这样可以保持
源文件的美观。

另外还有一个`ElementTree`的实现方式，

```python
def remove_xml_nodes(xml, l):
    """
    remove node in 'l' AND all comments,
    this is a bug of ElementTree of Python.
    """
    if not os.path.exists(xml):
        return
    tree = ET.parse(xml)
    root = tree.getroot()
    for name in l:
        # get parent node
        p_node = root.find('.//*[@name="' + name + '"]/..')
        if p_node:
            # get the node to be removed
            node = p_node.find('./*[@name="' + name + '"]')
            p_node.remove(node)
            # print('remove xml node: ', node.text)
    tree.write(xml, encoding='UTF-8', xml_declaration=True)
```

它的缺点是comments也被删除了，当然有很geek的解决方法。

(over)

[code_repo]: https://github.com/ybin/clean_unused_android_res