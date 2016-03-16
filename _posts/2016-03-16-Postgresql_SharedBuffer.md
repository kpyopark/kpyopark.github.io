---
published: false
layout: post
title: PostgreSQL에서 Shared_Buffer 접근하기
date: 2016-03-16T16:53:27.000Z
categories: postgres shared_buffer
---

## Shared Buffer 접근 제한 방법

### Pin
일종의 Reference Count역할을 합니다. Disk에 등록되어 있는 Buffer가 메모리로 올라올 경우, 이를 활용하는 Process는 무조건 Pin을 설정하여야 합니다. 백그라운드 Writer와 같이, 지속적으로 shared buffer를 비우려고 노력하는 독립 프로세스에게 해당 버퍼는 사용하는 중이니, 메모리에서 제거하지 말아 주세요라고 이야기하는 것입니다. 따라서, pin 되어 있지 않은 버퍼는 향후, 제거되거나, 다른 프로세스에서 다른 용도로 활용할 수 있기 때문에, 해당 버퍼 값을 바로 읽는 것은 매우 불안정한 행위가 됩니다. Pin을 얻기 위해서는 readBuffer 함수를 호출하면 Pin 카운트 값이 올라갑니다. 반대로 ReleaseBuffer를 수행하면, 해당 버퍼에 설정되어 있는 카운트가 줄어들게 됩니다. 따라서, 하나의 버퍼에 여러 프로세스가 동시에 접근하려고 할 경우, 핀 카운트는 1 이상의 값이 될 수 있습니다. 
일단, 트랜잭션을 벗어나면, Pin 카운트는 복원되어, 카운트 값이 줄어들게 됩니다.

### Content Lock
버퍼의 내용 변경을 판단하기 위하여, 두가지 형태의 Lock 형태를 제공합니다. 일단 Shared Lock입니다. Shared Lock은 읽기 전용 거래에서 활용됩니다. Shared Lock에 걸려있는 버퍼에 다른 Process가 다시 Shared Lock을 획득하려고 한다면, 중단없이 추가적인 Shared Lock을 획득할 수 있습니다. 다른 또 하나의 Lock은 Exclusive Lock입니다. Exclusive Lock의 경우, 해당 버퍼에 어떠한 Lock도 사전에 Locking되어 있지 않아야 합니다. 이미 Lock이 설정되어 있을 경우에는 사전 설정되어 있는 모든 Lock이 해제될 때까지 대기상태에 빠집니다. 모든 Lock이 설정 해제될 경우, exclusive Lock이 해당 버퍼에 설정되면서 이후 진행이 가능합니다. 일반적으로 READ Lock, Write Lock이라고도 불리웁니다. 

### 버퍼 접근에 대한 기본적인 Rule

1. 버퍼에 존재하는 Tuple을 검색하기 위해서는 해당 Tuple이 존재하는 버퍼에 Pin을 설정하고, Shared Lock/Exclusive Lock중에 하나를 설정하여야 합니다. 

2. 트랜잭션에서 특정 Tuple을 사용한 경우, 해당 Tuple이 존재하는 Buffer에 설정되어 있는 Read/Write Lock은 해제할 수 있지만, Pin은 해제하지 않습니다. 이럴 경우, Pin이 해제되어 있지 않기 때문에, 해당 Tuple이 메모리에서 사라지는 것을 방지할 수 있습니다. 다만, 일부 상태값이 변경될 수 있는데, 초기에 이미 가시성 결정(해당 Tuple이 보일지 말지)을 진행하였기 때문에, 이후 상태값이 변경되어도 큰 문제가 되지 않습니다.

3. 새로운 Tuple을 추가하거나, 기존 Tuple에 존재하는 XMIN/XMAX값을 변경하기 위해서는 해당 Tuple을 포함하는 버퍼에 pin과, exclusive lock을 설정하여야 합니다. exclusive Lock을 이용해서 다른 프로세스가 변경 도중에 값을 읽는 부분 변경 값 읽는 현상을 방지합니다.

4. Shared Lock(READ Lock)만 설정되어 있는 경우라도, 해당 Buffer에 있는 Tuple의 Commit Status값 변경하는 것은 괜찮습니다. 해당 Bit의 경우, 참조성 성격이 강해서 충돌로 인한 0 값 설정이 된다 하더라도 별 문제가 되지 않으며, Bit 연산이기 때문에, 충돌이 날 확률이 작습니다. 다만, HEAP_XMIN_INVALID, HEAP_XMIN_COMMITTED가 동시에 세팅이 되는 경우에는, WAL Log 상태이기 때문에, 해당 버퍼에 무조건 exclusive lock을 설정해주어야 합니다. 

5. 페이지에 올라와 있는 Tuple을 메모리에서 물리적으로 삭제하거나, 압축을 하기 위해서는, 해당 버퍼에 Pin과 exclusive Lock을 설정하고, 지속적으로 Pin 값이 1인 상태로 있는지 확인해야 합니다. Pin값이 1이란 이야기는 해당 Buffer를 다른 Backend 프로세스가 전혀 사용하지 않는 것이기 때문에, 안정적으로 Tuple에 대한 삭제, 조정을 수행할 수 있습니다. 




