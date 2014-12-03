title: PlantUML示例
date: 2014-12-03 11:08:36
categories: 工具
tags: [plantuml, UML]
---

PlantUML示例集合。

<!--more-->

##### 活动图（activity diagram）
```dot
@startuml activity-diagram.png
(*)--> if "  test A" then
    -->[true] if "  test B" then
        -->[true] if "  test C" then
            -->[true] (*)
        else
            -right->[false] "C fail"
            -->"finish"
        endif
    else
        -right->[false] "B fail"
        -->"finish"
    endif
else
    -right->[false] "A fail"
    -->"finish"
    -left-> (*)
endif
@enduml
```

##### 活动图(activity diagram, new syntax)
```dot
@startuml activity-diagram-new-syntax.png
scale 1.5
title How to lock an object in JamVM\n
start
#YellowGreen:read object's lockword;
if (object locked ?) then (yes)
    if (locked by myself ?) then (yes)
        if (lock too much times ?) then (yes)
            #YellowGreen:lock the monitor;
            #YellowGreen:inflate thin lock;
        else (no)
            #YellowGreen:add lock count;
        endif
    else (no)
        #YellowGreen:lock the monitor;
        while (is it a thin lock ?) is (yes)
            #YellowGreen:set FLC bit;
            if (try to lock) then (success)
                #YellowGreen:inflate thin lock;
            else (fail)
                #YellowGreen:wait on monitor;
            endif
        endwhile (no)
        #YellowGreen:while loop finished;
    endif
else (no)
    #YellowGreen:lock it and return;
endif
stop
@enduml
```

##### 类图（class diagram）
```dot
@startuml class-diagram.png
class Demo {
    int i = 1;
    int j = 2;
    
    +static void main()
}
@enduml
```

(to be continued)