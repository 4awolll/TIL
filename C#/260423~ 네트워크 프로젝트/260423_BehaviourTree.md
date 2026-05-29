# 1. FSM

## 기본 구조

```c++
switch(currentState)
{
    case State:Idle:
        // 동작
        break;
    case State:Move:
        // 동작
        break;
}
```
## Ex
= 상태(State)가 3개 일 때는 좋은 선택임
근데 만약 상태가 10개 이상이라면?

-> 상태수가 늘수록 코드 복잡도 폭증

# 2. State Pattern

## 구조

IState.cs(interface)

    - IdleState
    - MoveState
    - JumpState
  - StateMachine.cs

## 상태(State) 구성

- Idle
- Move
- Jump

+ 상태(State)
- Attack
- Dash
- Air 관련
- Crouch

"전이(Transition) 연결"

```
상태(State)가 많아질수록 
    - 비슷한 상태 간 조건중복이 많이 됨
```


# Behaviour Tree(BT)

> 루트 노드 -> 자식 노드를 순서대로 조건부 판단

Q. State Pattern과의 차이?
 - 상태(State)는 선택된 현재 상태(State)만 실행 -> 그 상태가 가진 조건부 판별만함
 - BT는 조건을 매 프레임 단위로 루트 노드에서부터 판단

## BT의 구조
```
Node들로 이루어져 있다.

public enum NodeState { Success, Failure, Running }

public abstract class Node
{
    public abstract NodeState Evaluate();
}
```

## 다룰 Node
- Selector
- Sequence
- Leaf (Condition/Action)

## Node 종류
1. Composite Node
   
- 자식 노드들의 실행 흐름을 제어한다.
   - Sequnce - And 구조
     - 순서대로 실행
     - 하나라도 실패하면 즉시 Failure 리턴
     - 모두 성공하면 Success 반환
   - Selector - Or 구조