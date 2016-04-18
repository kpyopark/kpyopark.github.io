---
published: true
layout: post
date: "2013-04-17 22:10:00 -0500"
categories: postgres lock manager
---
### Lock의 종류

spinlock. - 매우 짧은 순간의 Lock을 담당. 수십 Line 이상이 되는 명령어가 Lock이후에 사용되거나, kernel call이 호출되는 경우에는 spinlock을 사용하면 안됩니다. spinlock은 하드웨어서 제공하는 원자성이 있는 Test & Set 명령어를 이용해서 구현됩니다. spinlock을 가지고 있는 프로세스는 busy waiting(무한 for loop로 생각하면 될듯)을 수행합니다. 또한 일분 이상 Lock이 걸려있는 경우, 자동으로 타임아웃이 발생하게 되어있습니다. 이는 원래 매우 짧은 시간동안 락을 유지해야 되는 것이 원칙이기 때문에 분단위로 락이 잡혀있다는 것은 오류 상황이라는 반증이 됩니다. 

Lightweight Lock (LWLock) - 공유 메모리에 있는 공용 구조체를 접근하기 위해서 보통 사용하는 가벼운 형태의 Lock입니다. 배타적 모드와, 공유 모두 모두 지원하기 때문에, 상황에 마추어서 Read/Write 락을 이용하거나, Read-Only 락을 이용하면 됩니다. 실제 Lock 경합이 발생하는 경우, SysV에서 제공하는 Semaphore기능을 이용하기 때문에, Busy Wait 형태가 아닌 해당 프로세스 블락형태로 커널에서 처리됩니다. 

Regular Lock - 다양한 형태의 Lock 메카니즘을 제공합니다. 추가적으로 Deadlock 검증을 수행할 수 있습니다. 일반적인 사용자 요청에 의한 Lock의 경우 모두 Regular Lock을 이용하여 처리하여야 합니다. 

SIReadLock - 

* spinlock 또는 Lightweight Lock을 획득하려고 할 경우에 이미 잠근 Lock이 있는 경우에는, Query수행이 중단되고, die() 인터럽트가 발생합니다. Regular Lock의 경우 이와 같은 제약조건이 없습니다.

### Regular Lock의 조건
