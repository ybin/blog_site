title: clojure模块引用
date: 2015-02-12 16:42:59
categories: clojure
tags: [clojure, module]
---

clojure模块引用。

<!-- more -->

### 格式说明

函数语法：`(require (quote <prefix list>|<libspec>)+ <optional flag>*)`
`ns`格式： `'(:require [<prefix list>|<libspec>]+ <optional flag>*)`

`+`表示一个或者多个，`*`表示零个或者多个。`use`跟`require`是一样的，
他们两个只是在<libspec>的参数中有区别。`:use`, `:require`最终都会调用`load-libs`函数。

注意：prefix list是list(一般如此，但是并没有强制)，而libspec是symbol或者vector。

##### libspec

libspec指定一个lib的信息，包括lib name以及其他可选的选项，其格式如下：

- 一个全限定lib name(symbol)，如`'clojure.string`、`'clojure.set`等
- 一个vector：`[<lib name> <optional options>]`，如果libspec放在prefix list中，则<lib name>不能包含点号(`.`)。

如`[string :as str :refer [trim replace]]`、`[set :as s]`、`[clojure.string]`(不能放到prefix list里面)，
libspec支持的参数包括，

1. `:as`，即alias
2. `:refer`，加载模块，并将函数引入当前命名空间

如果是`use`的话还支持`refer`函数的参数，`use`自动将模块中的函数引入当前namespace，
即`use = require + refer`，

1. `:exclude`
2. `:only`
3. `:rename`

`use`的`:only`, `:exclude`, `rename`和`require`的`:refer`都是`clojure.core/refer`函数的参数。

##### prefix list

prefix list是一个list，它的格式是这样的： `'(<prefix> <libspec>+)`，
其中<prefix>指的是一个或多个lib路径的公共部分，如`clojure.string`和`clojure.set`他们的公共部分
为`clojure`，注意，此时libspec里的lib name不能包含点号(`.`)。
例如：`(clojure [set :as s] [string :as str :refer [trim replace]] walk)`

注意，1.7版本有一个bug，即prefix list允许没有libspec，如`(require '(clojure.set))`，这个表达式
可以正常运行，但是模块并没有被加载。

##### flag

flag包括：`:reload`，`:reload-all`，`:verbose`，这些flag是针对`require`、`use`的，
不是针对prefix list的，也不是针对libspec的，这一点儿一定要注意。

1. `:reload`，即使已经加载过模块也仍然再次加载，
2. `:reload-all`，除了包含`:reload`的行为之外，被加载的模块直接或者间接使用的模块也会重新加载，
3. `:verbose`，显示加载信息

例如：`(require '(clojure [string :as str] set) :reload-all)`

### 示例

```clojure
(require '(clojure set string)
         '(clojure [zip :as z] [template :as t])
         'clojure.pprint)
```

```clojure
(ns foo.bar
  (:require (clojure set string)
            (clojure [zip :as z] [template :as t])
            clojure.pprint))
```

```clojure
(use '(clojure (string :only (trim replace)) set)
     'clojure.walk)
```

```clojure
(ns foo.bar
    (:use (clojure (string :only (trim replace))
                   set)
           clojure.walk))
```

### Import Java modules

用`import`宏引入Java类作为clojure模块。

```clojure
(import '(java.util Date Calendar)
        'java.net.URI)
```

参数为Java类的全路径名，如果要引入同一个package中的多个类，则可以使用简略形式：
`'(package C1 C2 C3)`，这样就一次性引入这个package中的多个类了。

对应的`ns`参数格式为，

```clojure
(ns foo.bar
    (:import (java.util Date Calendar)
              java.net.URI))
```

(over)