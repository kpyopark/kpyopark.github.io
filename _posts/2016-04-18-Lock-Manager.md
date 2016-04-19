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

### Lock의 자료 구조

### Lock 종류에 따른 속성
#### LOCK Object

tag - 
	key field. lock hashtable에서 사용되는 Key 값. 

grantMask -
	현재 해당 Lock Object에 설정되어 있는 Lock Type이 어떤 것인지 표시해줌.
    
waitMask -
	대기상태에 있는 Lock의 종류를 보여줌

procLocks -
	Lock object에 할당된 모든 PROCLOCK을 소유하고 있는 공유 메모리 상의 Queue 목록
    
waitProcs -
	다른 Backed Process가 끝날때까지 대기하고 있는 Backend를 표시하는 PGPROC 객체 전체를 포함하는 공유 메모리 상의 Queue 목록
    
nRequested -
	해당하는 Lock Object가 몇 번의 Lock 요청을 받았는지 표시. 동일 프로세스에서 여러번 호출하는 경우에도 카운팅을 수행
    
requested -
	Lock 종류에 따라 시도한 횟수를 저장
    
nGranted -
	성공적으로 Lock을 획득한 횟수를 저장

granted -
	종류별로 현재 설정되어 있는 Lock의 갯수를 저장

위 내용을 보면, 
0 <= nGranted <= nRequested 관계가 성립됩니다. 추가적으로 
0 <= granted[i] <= requested[i] 또한 성립됩니다. 

만약, 모든 값이 0이 되는 경우 해당 Lock Object는 해제될 수 있습니다.

#### PROCLock Object

tag - 
	key field. PROCLock hashtable 에서 Key 역할을 합니다. 
    tag.myLock - PROCLock에 연결되어 있는 Lock 오브젝트의 포인터를 가지고 있습니다. 
    tag.myProc - PROCLock을 현재 가지고 있는 backend process의 PROC 포인터를 가지고 있습니다. 

holdMask - 
	PROCLOCK을 통해서 성공적으로 Lock을 획득한 Mode를 가지고 있음.
    LOCK object의 grantMask의 서브세이어야 하며, PGPROC 오브젝트가 가지고 있는 heldLocks의 서브셋이어야 함

releaseMask -
	LockReleaseAll 시에 해제되는 대상 Lock Mode를 표시합니다. 
    holdMask의 서브셋이어야 합니다. 

lockLink - 
	동일한 Lock에 대하여 PROCLOCK 오브젝트 Queue를 표시하는 포인터

procLink - 
	동일 Backend 프로세스에서 소유한 PROCLOCK에 대한 Queue를 표시하는 포인터

### Lock Manager의 내부 Locking

8.2 버젼 이전에서는 LockMgrLock이라는 단일 Lock을 이용하여 모든 잠금을 수행하였습니다.
이는 당연히 경합을 발생시켰고, 이를 수정하기 위해, 파티션이라는 개념을 도입하였습니다. 즉 특정 락이 관리하는 부분을 한정했다는 이야기 입니다. 하지만, 이로 인하여, 여러 파티션을 동시에 수정하는 경우도 발생했기 때문에, 내부적으로 락 객체들간의 동기화가 추가적으로 구현되어야 했습니다. 정리하면 아래와 같습니다.

* 특정 파티션에는 하나의 Lock이 할당됩니다. 할당되는 락은 LOCKTAG 값에 의해 Hastable에서 관리됩니다. 

* LOCKS, PROCLOCKS는 서로 다른 파티션에, 서로 다른 Hash 전략을 이용하여, 충돌없이 활용할 수 있습니다. 
  (이 부분은 dynahash.c 에 있는 "partitioned table"을 참고하시면 됩니다. - 확인해 볼 것)
  PROCLOCK의 경우 LOCK에서 추출한 Hash값의 low-bit가 동일할 경우에는 사용할 수 있게 됩니다. (proclock_hash 참고)

* 8.2이전의 경우, PGPROC은 한개의 PROCLOCKS 리스트만을 가지고 있었으나, 지금은 파티션으로 분리되어 있는 PROCLOCKS 리스트의 목록을 가지게 되었습니다. 이로 인하여, 특정 PROCLOCK에 접근하기 위해서는 먼저 LWLock(아마, PROCLOCKS 리스트 전체 목록에 대한 관리)을 획득하여야 합니다.

* 파티션에 대한 Lock을 획득하기 위해서는 데드록방지를 위하여, 파티션 넘버 순서에 따라 락을 획득하여야 합니다. 

* 백엔드 프로세스가 내부적으로 가지고 있는 LOCALLOCK 해쉬테이블의 경우, 파티션 처리를 하지 않고 있습니다. 

### Fask Path Locking

자주 사용되지만 충돌발생이 잘 안되는 특수한 경우 적용하기 위한 특별한 알고리즘

(1) 약한 테이블 락 : INSERT, UPDATE, DELETE, SELECT에서 사용하는 모든 테이블과 시스템 카탈로그에 대해서는 락을 획득해야 함. 대부분의 DML 수행시에는 동시에 처리할 수 있음. 하지만 DML의 경우(예. CLUSTER, ALTER TABLE, DROP 등등) 또는 LOCK TABLE과 같은 경우에는 DML을 위해서 획득한 Weak 락(AccessShareLock, RowShareLock, RowExclusiveLock)과 충돌이 발생함

(2) VXID 락 : 트랜잭션은 모두 자신만의 가상 트랜잭션 아이디에 대한 락을 가지고 있습니다. CREATE INDEX CONCURRENTLY 수행 중인 경우 또는 HOT STANDBY상태인 경우에 충돌이 발생합니다. 

락 충돌을 방지하기 위하여, 파티션 개념을 도입했지만, 짧은 쿼리가 매우 빈번하게 동일 테이블에 접근하는 경우에는 해당 테이블에 속한 파티션 락이 병목이 될 수 밖에 없습니다. 이런 현상은 2 코어 서버에서 조차, 매우 악명이 높았습니다. 

이를 완화하기 위하여, 9.2 버젼 이상에서는, 백엔드 프로세스 자체적인 PGPROC 구조 안에서 공유되지 않는 테이블에 대한 자체적인 락을 소유할 수 있게 지원하고 있습니다.

구현방법은 1024개의 정수 카운트 배열을 준비한 후, 1024개로 분할된 LOCK 공간에 맵핑하는 방법입니다. 이후, 각 파티션에 해당하는 비공유 테이블에 강한 락(ShareLock, ShareRowExclusiveLock, ExclusiveLock, and AccessExclusiveLock)이 발생하는 경우 카운트를 올려줍니다. 따라서 개별 카운트의 값이 0인지 아닌지를 확인하는 것으로 비공유 테이블에 락이 걸려있는지 없는지 여부를 확인할 수 있습니다. 












