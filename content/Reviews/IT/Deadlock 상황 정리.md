


# Table 간 데드락

mysql 데드락 문서에 예시로 나와있는 경우. `select for update` 후 서로 테이블을 참조하려 하면 데드락 감지된다.

# Upsert 시 데드락

Upsert batch 작업을 하는데, 두 배치 작업에서 공유하고 있는 pk가 있다면 발생 가능하다.

두 배치 트랜잭션이 pk1 과 pk2 를 모두 다른 순서로 포함하고 있다면
Tx1 이 pk1 락을 획득한 다음 pk2 락을 획득하려 하고
Tx2 는 pk2 락을 획득한 다음 pk1 락을 획득하려 하므로 데드락에 빠진다.

insert 문이라고 하더라도 pk 를 지정하는 순간 해당 pk 에 대한 락을 잡는 모양이다.

# Upsert 시 데드락 2

같은 pk 로 여러 트랜잭션들이 경쟁하는 경우 발생 가능하다. (매우 비직관적)

첫 트랜잭션이 pk 에 대한 x-lock 을 먼저 잡고, 다른 트랜잭션들은 기다리게 된다. 기다리는 트랜잭션이 둘 이상이면 데드락 가능성 생기게 된다.

mysql 만 그런지 모르겠는데 이 상황에서 대기 트랜잭션들이 기회를 잡아 락을 잡으려 할 때 s-lock -> x-lock 순서로 잡게 된다고 함 (아마 dup key 일 때 update 를 해야 하니까 x-lock 을 잡는 듯) 과정에서 순서가 안맞으면 x-lock 을 못잡게 되면서 데드락에 빠진다.

## s-lock, x-lock

x-lock 은 `select for update` 에서 거는 락이고, 다른 트랜잭션이 읽지도 못하게 만든다. 락 걸린 대상은 나만 독점적으로 보고 쓸 수 있게 된다. update, delete 될 때 걸리고 insert 시에도 pk 에 대해서 걸린다.

s-lock 은 다른 트랜잭션이 읽을 순 있게 한다. 읽기 잠금이라고 표시하기도 하는데 정말 끔찍하게 잘못 붙은 이름이라고 생각함. (읽지 못하게 만드는 것 같잖아)

s-lock 은 여러번 중첩되어 걸릴 수 있다. 하나라도 남아있다면 x-lock 을 걸 수 없고 업데이트 되지 않는다.



