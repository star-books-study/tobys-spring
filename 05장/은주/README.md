# 5장. 서비스 추상화
- 스프링이 어떻게 성격이 비슷한 여러 종류 기술을 추상화하고, 일관되게 사용할 수 있게 지원하는지 살펴보자

## 5.1. 사용자 레벨 관리 기능 추가
### 5.1.3. UserService.upgradeLevels()
- DAO 는 데이터를 어떻게 가져오고 조작할지를 다루는 곳이지, 비즈니스 로직을 두는 곳이 아니다.
- UserService는 UserDao 구현 클래스가 바뀌어도 영향받지 않도록 해야 한다.

### 5.1.5. 코드 개선
- 코드에 `중복` 은 없는가
- 코드가 `무엇을 하는 지` 이해하기 편리한가
- 코드가 자신이 있어야 할 자리에 있는가
- 앞으로 변경이 일어난다면 어떤 것이며, 변경에 쉽게 대응가능한가

#### upgradeLevels() 리팩토링
```java
public void upgradeLevels() {
  List<User> users : users) {
    if (canUpgradeLevel(user)) {
      upgradeLevel(user);
    }
  }
}

public boolean canUpgradeLevel(User user) {
  Level currentLevel = user.getLevel();
  switch (currentLevel) {
    case BASIC:
      return (user.getLogin() >= MIN_LOGCOUNT_FOR_SILVER);
    case SILVER:
      return (user.getRecommend() >= MIN_RECOMMEND_FOR_GOLD);
    case GOLD:
      return false;
    default:
      throw new IllegalArgumentException("Unknown Level: "+ currentLevel);
  }
}

private void upgradeLevel(User user) {
  if (user.getLevel() == Level.BASIC) 
    user.setLevel(Level.SILVER); 
  else if (user.getLevel() == Level.SILVER) 
    user.setLevel(Level.GOLD); 
  userDao.update(user);
}
```
- 하지만 upgradeLevel() 메소드는 여러 문제점이 존재한다
  - 다음 레벨이 무엇인지에 대한 로직과 사용자 오브젝트 level 필드를 변경하는 로직이 함께 존재한다
  - 잘못된 레벨이 들어온 상황에 대한 예외 처리가 없다  
- 레벨의 순서와 `다음 단계 레벨이 무엇인지 결정`하는 일은 level 에게 맡긴다
```java
public enum Level {
  GOLD(3, null), SILVER(2, GOLD), BASIC(1, SILVER);
  private final int value; 
  private final Level next; // 다음 단계 레벨 표현
  ...
}
```
- 사용자 오브젝트의 level 필드를 변경하는 로직은 User 에게 맡긴다
  - UserService 가 일일이 업그레이드 시 User의 어떤 필드를 수정한다는 로직을 갖고 있기 보다는 User 에게 정보를 변경하라고 요청하는 것이 낫다
```java
// User 객체의 메소드
public void upgradeLevel() {
  Level nextLevel = this.level.nextLevel(); 
  if (nextLevel == null) {
    throw new IllegalStateException(...);
  }
  else {
    this.level = nextLevel;
  }
}
```
- User에 업그레이드 작업을 담당하는 **독립적인 메소드를 두고 사용할 경우, 업그레이드 시 기타 정보가 필요할 경우 유용하다**
  - ex) 가장 최근에 레벨 변경 일자를 User에 저장하고 싶은 경우
```java
// AS-IS
private void upgradeLevel(User user) {
  if (user.getLevel() == Level.BASIC) 
    user.setLevel(Level.SILVER); 
  else if (user.getLevel() == Level.SILVER) 
    user.setLevel(Level.GOLD); 
  userDao.update(user);
}

// TO-BE
private void upgradeLevel(User user) {
  user.upgradeLevel();
  userDao.update(user);
}
```
- 객체지향적인 코드는 다른 오브젝트의 데이터를 가져와서 작업하는 대신, **데이터를 갖고 있는 다른 오브젝트에게 작업을 해달라고 요청** 하는 것이다
  - 오브젝트에게 데이터를 요구하지 말고, `작업을 요청하라` 는 것이 객체지향 프로그래밍의 가장 기본이 되는 원리이다

## 5.2. 트랜잭션 서비스 추상화
- 사용자 레벨 관리 작업 수행 도중에 문제가 생긴다면, 그때까지 진행된 변경 작업도 모두 취소시키도록 결정했다

### 5.2.1. 모 아니면 도
- 4번째의 사용자를 처리하다가 예외 발생하여 작업 중단됐으니, 이미 레벨이 수정된 사용자도 원래 상태로 돌아가는 것을 예상했다

#### 테스트 실패의 원인
- 모든 사용자의 레벨을 업그레이드 하는 upgradeLevels() 메소드가 **하나의 트랜잭션 안에서 동작하지 않았기 때문** 이다

### 5.2.2. 트랜잭션 경계설정