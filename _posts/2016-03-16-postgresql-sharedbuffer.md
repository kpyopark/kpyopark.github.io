---
layout: post
title: PostgreSQL에서 Shared_Buffer 접근하기
date: '2013-11-29 00:00:00 -0500'
categories: postgres shared_buffer
published: true
---

# Shared Buffer 접근 제한 방법

## Pin

일종의 Reference Count역할을 합니다. Disk에 등록되어 있는 Buffer가 메모리로 올라올 경우, 이를 활용하는 Process는 무조건 Pin을 설정하여야 합니다. 백그라운드 Writer와 같이, 지속적으로 shared buffer를 비우려고 노력하는 독립 프로세스에게 해당 버퍼는 사용하는 중이니, 메모리에서 제거하지 말아 주세요라고 이야기하는 것입니다. 따라서, pin 되어 있지 않은 버퍼는 향후, 제거되거나, 다른 프로세스에서 다른 용도로 활용할 수 있기 때문에, 해당 버퍼 값을 바로 읽는 것은 매우 불안정한 행위가 됩니다. Pin을 얻기 위해서는 readBuffer 함수를 호출하면 Pin 카운트 값이 올라갑니다. 반대로 ReleaseBuffer를 수행하면, 해당 버퍼에 설정되어 있는 카운트가 줄어들게 됩니다. 따라서, 하나의 버퍼에 여러 프로세스가 동시에 접근하려고 할 경우, 핀 카운트는 1 이상의 값이 될 수 있습니다. 일단, 트랜잭션을 벗어나면, Pin 카운트는 복원되어, 카운트 값이 줄어들게 됩니다.

## Content Lock

버퍼의 내용 변경을 판단하기 위하여, 두가지 형태의 Lock 형태를 제공합니다. 일단 Shared Lock입니다. Shared Lock은 읽기 전용 거래에서 활용됩니다. Shared Lock에 걸려있는 버퍼에 다른 Process가 다시 Shared Lock을 획득하려고 한다면, 중단없이 추가적인 Shared Lock을 획득할 수 있습니다. 다른 또 하나의 Lock은 Exclusive Lock입니다. Exclusive Lock의 경우, 해당 버퍼에 어떠한 Lock도 사전에 Locking되어 있지 않아야 합니다. 이미 Lock이 설정되어 있을 경우에는 사전 설정되어 있는 모든 Lock이 해제될 때까지 대기상태에 빠집니다. 모든 Lock이 설정 해제될 경우, exclusive Lock이 해당 버퍼에 설정되면서 이후 진행이 가능합니다. 일반적으로 READ Lock, Write Lock이라고도 불리웁니다.

## 버퍼 접근에 대한 기본적인 Rule

1. 버퍼에 존재하는 Tuple을 검색하기 위해서는 해당 Tuple이 존재하는 버퍼에 Pin을 설정하고, Shared Lock/Exclusive Lock중에 하나를 설정하여야 합니다.
2. 트랜잭션에서 특정 Tuple을 사용한 경우, 해당 Tuple이 존재하는 Buffer에 설정되어 있는 Read/Write Lock은 해제할 수 있지만, Pin은 해제하지 않습니다. 이럴 경우, Pin이 해제되어 있지 않기 때문에, 해당 Tuple이 메모리에서 사라지는 것을 방지할 수 있습니다. 다만, 일부 상태값이 변경될 수 있는데, 초기에 이미 가시성 결정\(해당 Tuple이 보일지 말지\)을 진행하였기 때문에, 이후 상태값이 변경되어도 큰 문제가 되지 않습니다.
3. 새로운 Tuple을 추가하거나, 기존 Tuple에 존재하는 XMIN/XMAX값을 변경하기 위해서는 해당 Tuple을 포함하는 버퍼에 pin과, exclusive lock을 설정하여야 합니다. exclusive Lock을 이용해서 다른 프로세스가 변경 도중에 값을 읽는 부분 변경 값 읽는 현상을 방지합니다.
4. Shared Lock\(READ Lock\)만 설정되어 있는 경우라도, 해당 Buffer에 있는 Tuple의 Commit Status값 변경하는 것은 괜찮습니다. 해당 Bit의 경우, 참조성 성격이 강해서 충돌로 인한 0 값 설정이 된다 하더라도 별 문제가 되지 않으며, Bit 연산이기 때문에, 충돌이 날 확률이 작습니다. 다만, HEAP\_XMIN\_INVALID, HEAP\_XMIN\_COMMITTED가 동시에 세팅이 되는 경우에는, WAL Log 상태이기 때문에, 해당 버퍼에 무조건 배타적 잠금을 설정해주어야 합니다.
5. 페이지에 올라와 있는 Tuple을 메모리에서 물리적으로 삭제하거나, 압축을 하기 위해서는, 해당 버퍼에 Pin과 배타적 잠금을 설정하고, 지속적으로 Pin 값이 1인 상태로 있는지 확인해야 합니다. Pin값이 1이란 이야기는 해당 Buffer를 다른 Backend 프로세스가 전혀 사용하지 않는 것이기 때문에, 안정적으로 Tuple에 대한 삭제, 조정을 수행할 수 있습니다.

## Contents Lock의 변화

7.x 버젼 이전까지 공유 버퍼 관리자가 단 한개의 시스템 잠금\(Lock\) 메커니즘을 가지고 동작하였습니다. 이로 인해서 공유 버퍼에서 발생하는 잠금 경합이 문제가 되었습니다.

8.1 버젼에서는 여러개의 Lock을 구분하여 생성하였습니다. 먼저, 시스템 전반적인 Lock을 이용하고자 할때 사용하는 **BufMappingLock**입니다. 말 그대로, Shared Buffer와 해당 Buffer Page를 지칭하는 Handle\(또는 tag\)와의 Mapping 테이블 자체에 Lock을 설정하는 방법입니다. 현재 메모리에 버퍼가 있는 여부를 검사하기 위해서는 BufMappingLock에 단순하게 Shared Lock을 설정하는 것으로 충분합니다. 하지만, Buffer에 새로운 페이지를 할당하기 위해서는 Exclusive Lock을 설정하여야 합니다.

**8.2 버전으로 올라오면서 해당 BufMappingLock은 다수의 파티션 Lock으로 바뀌었습니다**. 즉 Shared Buffer 공간 전체를 관리하는 것에서 일정 부분의 파티션 Buffer Pool만 관리하도록 하여 경합을 최소화하도록 변경되었습니다. 여러개의 Lock을 동시에 사용하는 경우에 발생할 수 있는 데드락 방지를 위하여 BufMappingLock 인스턴스는 파티션별로 우선순위를 부여하고, 우선순위 별로 Lock을 수행하도록 구성해야 합니다. \(데드락은 서로 다른 Lock이 서로 다른 순서로 동작하는 도중에 발생합니다. 따라서 개별 Lock에 호출 우선순위를 부여하고, 우선순위별로 Lock 잠금 및 해제를 수행하면 데드락은 발생하지 않습니다.\)

spin lock의 일종인 **buffer\_strategy\_lock**이 추가로 제공됩니다. 해당 Lock은 Shared 버퍼의 빈공간 리스트에 접근하거나, 버퍼를 대치하기 위하여 검색을 할 경우에 활용됩니다. spin lock 특성상 짧은 시간동안의 exclusive lock을 제공하기 위한 방법이기 때문에, 나중에 위 두가지 Operation\(빈공간 리스트 접근 및 버퍼 리스트 접근\)에 왜 spin lock을 적용하는지 확인해 보도록 하겠습니다.

각 **Shared 버퍼의 헤더에는 스핀 락**이 존재합니다. 헤더 값을 조회하거나 헤더 값을 변경하고자 할 경우, 전역 Lock 메커니즘을 활용하지 않고, 헤더에만 spin lock을 설정하여, 큰 경합 없이 버퍼별 헤더값 조회를 수행할 수 있습니다.

또한 **Shared 버퍼자체에는 Content자체에 대한 Lock**을 가지고 있습니다. 해당 Lock은 위해서 설명한 Content Lock과 정확하게 대응하며, 그 사용규칙은 위의 1번에서 5번까지 설명하였습니다.

마지막으로 버퍼별 락, 일명 io\_in\_progress 락이 존재합니다. 해당 락은 버퍼의 내용을 채우기 위한 I/O 가 발생할 경우, 이를 기다려주는 역할을 합니다.

## 일반적인 버퍼 할당 전략

먼저 "사용하지 않는" 버퍼 리스트가 존재합니다. "사용하지 않는" 다는 의미는 아예 사용한 적이 없어서 해당 버퍼에 아무값도 없는 경우를 의미합니다. 만약 단 하나의 조회를 위해서 이미 할당된 적이 있는 공유 버퍼의 경우, 핀이나, 잠금이 설정되어 있지 않을 경우, 사용하지 않는 버퍼 리스트로 돌아갈 수 있지만, 여기에서는 아예 사용된 적 없는 버퍼 리스트가 있다고 생각하면 됩니다.

이 "사용하지 않는" 버퍼 리스트를 프리 리스트\(Free List\)라고 호칭하겠습니다.

프리 리스트는 어는 경우에서나 제일 처음 활용될 수 있는 대상 버퍼가 됩니다. 프리 리스트는 전역에서 모두 접근할 수 있습니다만, 먼저, buffer\_strategy\_lock을 활용하여 잠금 설정을 수행하여야 합니다.

프리 버퍼 리스트가 비어 있어, 새로운 버퍼 할당을, 기존 프리 리스트에서 가지고 오지 못할 경우, 이제는 현재 사용중인 버퍼에서 재활용할 대상을 선정하여야 합니다. 이때 다음과 같은 시계-제거\(Clock Sweep\) 알고리즘을 적용합니다.

개별 버퍼에는 위에서 예기한 사용 카운터가 있습니다. 백엔드 프로세스에서 핀을 설정하면, 사용 카운터가 올라가는데요. 사용 카운터를 헤더에서 읽어 오는 것은 거의 비용이 들지 않습니다.

일단, 전체 버퍼에서 사용 대상 목록을 작성합니다. 이를 재활용 목록이라고 부르도록 하겠습니다.\(실제로는 희생자 목록이라고 불리웁니다.\) 사용한 모든 버퍼는 이 재활용 목록에 포함되어야 합니다. 두번째는 재활용 대상을 결정하는 인덱스 값을 결정합니다. 이 인덱스 값 결정 로직이 아래와 같습니다.

1. 먼저 buffer\_strategy\_lock을 획득합니다. 
2. 프리 리스트가 비어 있지 않다면, 제일 앞단의 버퍼를 활용하기 위해서 프리 리스트에서 제거하고, 이를 돌려줍니다. 이후 buffer\_stategy\_lock을 제거합니다.
3. 프리 리스트가 존재하지 않을 경우, 재활용 목록에서 다음 재활용 목록을 지정하고, 그 다음 항목으로 시계 바늘\(인덱스\)을 옮김니다. 이후 buffer\_stategy\_lock을 제거합니다. 
4. 만약, 해당 버퍼가 이미 핀 설정 또는 락 설정이 되어 있을 경우, 프리 버퍼로 재활용 할 수 없기 때문에, buffer\_stategy\_lock을 재설정하고, 3번 항목으로 다시 돌아갑니다. 
5. 비어 있거나, 또는 재활용 가능한 버퍼를 돌려받은 경우, 해당 버퍼에 핀을 설정합니다.

## 링버퍼 할당 전략

대규모로 일회성 버퍼가 할당이 되는 경우가 있습니다. VACUUM이나, 대규모 Sequence scan이 발생한 경우인데요. 이럴 경우, 위와 같은 일반적인 시계-제거 알고리즘을 이용하는 대신에 조그마한 링 버퍼를 단일 버퍼 대신에 할당하여 활용합니다. 순차 조회시에 할당되는 링버퍼의 크기는 256KB 정도로 CPU의 L2 캐쉬에 충분히 들어갈 수 있도록 작게 만들어짔습니다. 대부분의 경우, 256KB의 링 버퍼를 다 사용하지는 않지만, 모든 경우에 적용을 할 수 있을 정도로 충분하게 링버퍼를 할당해야 합니다. 만약, 링 버퍼가 더티 상태로 설정되어 있을 경우에는 재활용 하기 위해서 WAL 로그를 쌓아야 합니다. 읽기 전용 순차 읽기의 경우에는 링버퍼의 효율이 가장 커지고, 만약 대량의 삽입, 수정이 발생하는 경우에는 일반적인 시계-제거 알고리즘과 차이가 없게 됩니다. Vacuum의 경우에도 순차 읽기와 동일하게 256KB의 링버퍼를 할당합니다. 다른 점은, Dirty표시되어 있는 페이즈를 링 버퍼에서 제거하지 않고, Flush를 시킨다는 점이 다릅니다. 링버퍼가 도입되기 전 8.3 버젼에서는 Vacuum된 버퍼를 프리리스트로 보내기 위하여, 바로바로 Flush를 시켰기 때문에, 과도한 Flush 작업이 발생되곤 했습니다. 대량 삽입의 경우, 링 사이즈가 16MB\(전체 shared buffer의 1/8을 넘지 않는 선에서\)를 할당합니다.

## 백그라운드 Writer 처리방식

백그라운드 Writer는 버퍼의 재활용성을 높이기 위해서, 활성화된 프로세스와는 별도로 동작하는 프로세스입니다. Checkpoint 순간에 모든 Dirty 버퍼를 Disk에 쓰기 시도를 진행합니다. 이를 위해서는 해당 버퍼가 더티되어 있는지 여부를 확인해야 합니다. 백드라운드 Writer는 버퍼 리스트에 어떠한 변환도 수행하지 않고, 또한 현재 해당 버퍼가 사용하는지 여부와는 전혀 무관하게, 더티 여부만을 확인하고 이를 Disk에 반영하기 때문에, buffer\_strategy\_lock에 접근할 필요가 없으며, 단순하게 Content Lock중 Shared Lock만을 설정하기 때문에 다른 Read 프로세스에 영향을 주지 않습니다.

