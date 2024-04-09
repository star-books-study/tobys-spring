# 2장. 테스트
- 스프링의 핵심인 IoC 와 DI 는 `오브젝트 설계와 생성, 관계, 사용` 에 관한 기술이다
- 스프링은 복잡한 엔터프라이즈 애플리케이션을 효과적으로 개발하기 위한 기술이다
  - 복잡한 어플리케이션 개발하는데 필요한 도구 중 하나는 `객체지향 기술`
  - 또 다른 도구는 강조하고 있는 것은 `테스트` 
- 복잡해지는 애플리케이션에 대응하는 2가지 전략
  - **확장과 변화를 고려한 객체지향적 설계** + 그것을 효과적으로 담을 수 있는 IoC/DI 같은 기술
  - **테스트 기술** ; 만들어진 코드를 확신하게 해주며 변화에 유연하게 대처할 수 있는 자신감을 주는 기술

## 2.1. UserDaoTest 다시 보기
### 2.1.1. 테스트의 유용성
- 테스트란 결국 내가 예상하고 의도한 대로 코드가 정확히 동작하는 지 확인해서, **만든 코드를 확신할 수 있게** 해준다

### 2.1.2. UserDaoTest 의 특징
#### 웹을 통한 DAO 테스트 방법의 문제점
- DAO 뿐만 아니라 서비스 클래스, 컨트롤러, JSP 뷰 등 모든 레이어의 기능을 다 만들어야 테스트가 가능하다
- 다른 계층의 코드와 컴포넌트, 심지어 서버의 설정 상태까지 모두 테스트에 영향을 줄 수 있으므로, 이런 방식으로 테스트하는 것은 번거롭고, 오류가 있을 때 빠르고 정확하게 대응하기가 힘들다.

#### 작은 단위의 테스트
- 테스트는 가능하면 **작은 단위로 쪼개서** 집중해서 할 수 있어야 한다.
- 테스트의 관심이 다르다면 테스트할 대상을 분리하고 집중해서 접근해야 한다.
- `단위 테스트(unit test)` : 작은 단위의 코드에 대해 테스트를 수행한 것
>  DB가 사용되어도 단위테스트인가?
>  - 어떤 개발자는 테스트 중에 DB가 사용되면 단위 테스트가 아니라고도 한다. 
>  - 그럼 UserDaoTest는 단위 테스트가 아니라고 봐야 할까?
> 
> NO!
> - 지금까지 UserDaoTest를 수행할 때 매번 테이블의 내용을 비웠다.
> - **사용할 DB의 상태를 테스트가 관장하고 있다면** 단위 테스트라고 해도 된다. <br>
> <br>
> 다만, **통제할 수 없는 외부의 리소스에 의존하는 테스트**는 단위 테스트가 아니라고 보기도 한다.
- 단위 테스트를 하는 이유
  - 개발자가 설계하고 만든 코드가 **원래 의도대로 동작하는지 개발자 스스로 빨리 확인받기 위해서**
  - 확인의 대상과 조건이 간단하고, 명확할수록 좋다

#### 자동수행 테스트 코드
- 테스트는 자동으로 수행되도록 **코드로 만들어지는 것이 중요** 하다
- 자동으로 수행되는 테스트의 장점은 자주 `반복` 할 수 있다는 것이다
- 애플리케이션을 구성하는 클래스 안에 테스트 코드를 포함시키는 것보다는 **별도로 테스트용 클래스를 만들어서** 테스트 코드를 넣는 편이 낫다.

#### 지속적인 개선과 점진적인 개발을 위한 테스트
- 단순 무식한 방법으로 정상동작하는 코드를 만들고, 테스트를 통해 매우 작은 단계를 거쳐가면서 계속 코드를 개선할 수 있다
- 기존 기능 뿐만 아니라 새로운 기능을 추가할 때도 기대한 대로 동작하는 지 확인할 수 있다

### 2.1.3. UserDaoTest 의 문제점
```java
public class XmlUserDaoTest {
    public static void main(String[] args) throws SQLException, ClassNotFoundException {
        ApplicationContext applicationContext = new GenericXmlApplicationContext("spring/applicationContext.xml");
        UserDao userDao = applicationContext.getBean(UserDao.class);

        User user = new User();
        user.setId("12341234");
        user.setName("제이크22522");
        user.setPassword("jakejake");

        userDao.add(user);

        System.out.println(user.getId() + " register succeeded");

        User user2 = userDao.get(user.getId());
        System.out.println(user2.getName());
        System.out.println(user2.getPassword());

        System.out.println(user2.getId() + " query succeeded");
    }
}
```
- 수동 확인 작업의 번거로움
  - add()에서 User 정보를 DB에 등록하고, 이를 다시 get()을 이용해 가져왔을 때 입력한 값과 가져온 값이 일치하는지를 테스트 코드는 확인해주지 않는다.
- 실행 작업의 번거로움
  - 아무리 간단히 실행 가능한 main() 메소드라고 하더라도 매번 그것을 실행하는 것은 제법 번거롭다.
  - 만약 DAO가 수백 개가 되고 그에 대한 main() 메소드도 그만큼 만들어진다면 전체 기능을 테스트해보기 위해 main() 메소드를 수백 번 실행해야 한다.

## 2.2. UserDaoTest 개선
### 2.2.1. 테스트 검증의 자동화
- 모든 테스트는 성공과 실패의 두 가지 결과를 가질 수 있다.
  -  테스트 에러 : 테스트가 진행되는 동안에 `에러가 발생`해서 실패
  -  테스트 실패 : 테스트 작업 중에 에러가 발생하진 않았지만 그 `결과가 기대한 것과 다르게` 나옴
- 테스트 프레임워크를 통해 두 경우 모두 검증할 수 있다.

### 2.2.2. 테스트의 효율적인 수행과 결과 관리
#### JUnit 테스트로 전환
- 프레임워크는 개발자가 만든 클래스에 대한 제어 권한을 넘겨받아서 **주도적으로 애플리케이션의 흐름을 제어**한다.
- 개발자가 만든 클래스의 오브젝트를 생성하고 실행하는 일은 프레임워크에 의해 진행된다.
- 따라서 프레임워크에서 동작하는 코드는 main() 메소드도 필요 없고 오브젝트를 만들어 실행하는 코드도 필요 없다.

#### 테스트 메소드 전환
- 기존에 만들었던 main() 메소드 테스트는 제어권을 직접 가졌기에 프레임워크에 적용하기엔 적합하지 않다. 
- 테스트 코드를 main()에서 일반 메소드로 옮겨야 한다.
- JUnit 테스트 메소드 요구조건
  - 메소드가 public으로 선언돼야 한다.
  - 메소드에 @Test 애노테이션을 붙여줘야 한다.

#### 검증 코드 전환
- JUnit은 예외가 발생하거나 assertThat()에서 실패하지 않고 테스트 메소드의 실행이 완료되면 테스트가 성공했다고 인식한다.

#### JUnit 테스트 실행
- 테스트 에러 : JUnit은 assertThat()을 이용해 검증을 했을 때 기대한 결과가 아니면 이 AssertionError를 던진다
- 테스트 예외 : 테스트 수행 중에 일반 예외가 발생한 경우에도 테스트 수행은 중단되고 테스트는 실패한다
```java
public class UserDaoTest {
    @Test
    public void addAndGet() throws SQLException {
        ApplicationContext applicationContext = new GenericXmlApplicationContext("spring/applicationContext.xml");

        UserDao userDao = applicationContext.getBean(UserDao.class);

        User userToAdd = new User();
        userToAdd.setId("hunch");
        userToAdd.setName("헌치");
        userToAdd.setPassword("password");

        userDao.add(userToAdd);

        User userToGet = userDao.get("hunch");

        Assertions.assertEquals(userToAdd.getId(), userToGet.getId());
        Assertions.assertEquals(userToAdd.getName(), userToGet.getName());
        Assertions.assertEquals(userToAdd.getPassword(), userToGet.getPassword());
    }
}
```