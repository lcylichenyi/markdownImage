

```flow
start=>start: 开始
end=>end: 结束，完成上锁
record=>operation: 记录当前G状态
park=>operation: 排队，挂起
iter=>operation: 自旋次数+1
wake=>operation: 被唤醒
clearSpin=>operation: 自旋数清零

isLocked=>condition: 有无锁
spin=>condition: 是否自旋
getLock=>condition: 是否抢到锁
isHungryOrLocked=>condition: 当前锁是否上锁或饥饿
isHungry=>condition: 当前锁是否饥饿


start->isLocked

isLocked(yes)->spin
isLocked(no)->end


spin(yes)->iter(left)->spin
spin(no)->record->getLock

getLock(yes)->isHungryOrLocked
getLock(no)->spin

isHungryOrLocked(yes)->park->wake->isHungry
isHungryOrLocked(no)->end

isHungry(yes,right)->end
isHungry(no)->clearSpin(left)->spin



```



