title: clojure模块引用
date: 2015-02-12 16:42:59
categories: clojure
tags: [clojure, module]
description: clojure, module, import, require, use
---

clojure模块引用。

<!-- more -->

注意：参数使用vector和list均可！

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

### Load Clojure modules

用`require`宏来加载clojure模块。

```clojure
(require '(clojure set string)
         '(clojure [zip :as z] [template :as t])
         'clojure.pprint)
```

参数为clojure文件的全路径名，如果要引入同一个目录下的多个clojure文件(模块)，
可以使用简略形式：`'(clojure set string)`，并且还可以在引入的同时给模块起别名，
`'(clojure [zip :as z] [template :as t])`。`require`只是加载模块，它并不会把
模块里的函数加到当前namespace里面来，所以访问的时候需要使用全限定名：
`(clojure.string/trim " abc ")`，如果起别名的话直接用别名访问即可。

对应的`ns`参数格式为，

```clojure
(ns foo.bar
  (:require (clojure set string)
            (clojure [zip :as z] [template :as t])
            clojure.pprint))
```

这样会加载整个模块里的所有函数，如果只需要部分函数呢？那就要使用`:refer`关键字，

```clojure
(require '(clojure.string :as str :refer (trim replace)) :reload-all)
```

这样就只会加载clojure.string模块里面的trim, replace两个函数。但是`:reload-all`是
什么？它是一个标志位，意思是加载clojure.string模块时，重新加载它，即使以前已经加载
过了，也要重新加载，并且clojure.string所依赖的模块也要重新加载，对比`:reload`，它
只是重新加载clojure.string而不去重新加载clojure.string所依赖的模块。

注意，用`:refer`加载的函数会自动添加到当前namespace，直接使用函数名即可，不用全限定名。

### use clojure modules

`use`自动将模块中的函数引入当前namespace，即`use = require + refer`，require负责加载，
refer负责引入当前namespace。`use`可以接收额外参数：`:only`, `:exclude`, `:rename`，
这些参数跟`clojure.core/refer`的参数是一样的。

```clojure
(use '(clojure (string :only (trim replace)) set)
     'clojure.walk)
```

以及`ns`参数格式，

```clojure
(ns foo.bar
    (:use (clojure (string :only (trim replace))
                   set)
           clojure.walk))
```

(over)