title: clojure的Var和Namespace在内存中的布局
date: 2015-02-03 11:09:10
categories: clojure
tags: [clojure, java]
---

Clojure中，Var和Namespace的内存布局。

<!-- more -->

### Layout of Var

```java
/** Layout of Var:

 每个线程都有自己的一套形如下图的Var存储布局，即这套内容是ThreadLocal的。
 
                                                        ThreadLocal<Frame> dvals
                                                                   | 
                   TOP                                             v
               +----------+       +----------+                +----------+
               | null     | <---- | prev     | <--- .... <--- | prev     |         
               +----------+       +----------+                +----------+         
               | bindings |       | bindings |                | bindings |         
               +----------+       +----------+                +----------+         
                    |                  |                           |                
                    |                  |                           |                 
                    v                  v                           v                 
 PersistentHashMap.EMPTY       +-------+--------+          +-------+--------+   
                               | var_0 | thread |          | var_0 | thread |   
                               |       | val_0  |          |       | val_0  |   
                               +-------+--------+          +-------+--------+   
                               | var_1 | thread |          | var_1 | thread |   
                               |       | val_1  |          |       | val_1  |   
                               +-------+--------+          +-------+--------+   
                               | var_2 | thread |          | var_2 | thread |   
                               |       | val_2  |          |       | val_2  |   
                               +-------+--------+          +-------+--------+   
                               |    .......     |          | var_3 | thread |   
                               +----------------+          |       | val_3  |   
                                                           +-------+--------+   
                                                           |    .......     |   
                                                           +----------------+
 */
```

### Layout of Namespace

```java
/** Layout of Namespace:

namespace 是全局的，mappings, aliases是依附于单独Namespace对象的。
 
                                                    mappings       
                                                    
                                               map: Symbol -> Var
                                      +-----> +--------+---------+  
                                      |       | quote  | Var obj |  
                                      |       +------------------+  
                                      |       | vector | Var obj |  
                                      |       +------------------+  
                                      |       | cons   | Var obj |  
                                      |       +--------+---------+  
            namespace                 |       |   . . . . . .    |
                                      |       +------------------+  
     map: Symbol -> Namesapce         |                            
 +--------------+---------------+     |                            
 | clojure.core | Namespace ojb | ----+  +--> +--------+---------+  
 +--------------+---------------+        |    | var_0  | Var obj |  
 | user         | Namespace obj | -------+    +--------+---------+  
 +--------------+---------------+             | var_01 | Var obj |  
 |                              |             +--------+---------+  
 |         . . . . . .          |             | var_02 | Var obj |  
 |                              |             +--------+---------+  
 +--------------+---------------+             |   . . . . . .    |  
 | other nsname | Namespace obj | ------+     +------------------+  
 +--------------+---------------+       |                           
                                        +---> +--------+---------+  
                                              | var_0  | Var obj |  
                                              +--------+---------+  
                                              | var_01 | Var obj |  
                                              +--------+---------+  
                                              | var_02 | Var obj |  
                                              +--------+---------+  
                                              |   . . . . . .    |  
                                              +------------------+  
            aliases              
                                 
     map: Symbol -> Namesapce    
 +--------------+---------------+
 | alias_0      | Namespace ojb |
 +--------------+---------------+
 | alias_1      | Namespace obj |
 +--------------+---------------+
 |                              |
 |         . . . . . .          |
 |                              |
 +--------------+---------------+
 | alias_n      | Namespace obj |
 +--------------+---------------+

 */
```