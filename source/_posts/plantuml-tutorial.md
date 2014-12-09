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
JamVM中对象锁的锁定流程。

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
UML中类之间的关系：依赖、关联、聚合、组合、继承、实现。

```dot
@startuml class-diagram.png
scale 1.5
/'组合关系(composition)'/
class Human {
    - Head mHead;
    - Heart mHeart;
    ..
    - CreditCard mCard;
    --
    + void travel(Vehicle vehicle);
}

Human *-up- Head : contains >
Human *-up- Heart : contains >

/'聚合关系(aggregation)'/
Human o-left- CreditCard : owns >

/'依赖关系(dependency)'/
Human .down.> Vehicle : dependent

/'关联关系(association'/
Human -down-> Company : associate

/'继承关系(extention)'/
interface IProgram {
    + void program();
}
class Programmer {
    + void program();
}
Programmer -left-|> Human : extend
Programmer .up.|> IProgram : implement
@enduml
```

(to be continued)