# 2장. 테스트
- 스프링이 개발자에게 제공하는 가장 중요한 가치 : 객체지향과 테스트
- 스프링으로 개발을 하면서 테스트를 만들지 않는다면 이는 스프링이 지닌 가치의 절반을 포기하는 셈
- 개발자들이 낭만이라고도 생각하는 눈물 젖은 커피와 함께 며칠간 밤샘을 하며 오류를 잡으려고 애쓰다가 전혀 생각지도 못했던 곳에서 간신히 찾아낸 작은 버그 하나의 추억이라는 건, 사실 진작에 충분한 테스트를 했었다면 쉽게 찾아냈을 것을 미루고 미루다 결국 커다란 삽질로 만들어버린 어리석은 기억일 뿐이다.
- 일반적으로 테스트하기 좋은 코드가 좋은 코드일 가능성이 높다. 그 반대도 마찬가지다.

## 2.1 UserDaoTest 다시 보기
### 2.2.1 테스트의 유용성
테스트란 결국 내가 예상하고 의도했던 대로 코드가 정확히 동작하는지를 확인해서, 만든 코드를 확신할 수 있게 해주는 작업이다.
테스트가 실패한 후 코드의 결함을 제거해가는 작업, 일명 디버깅을 거치게 되고, 결국 최종적으로 테스트가 성공하면 모든 결함이 제거됐다는 확신을 얻을 수 있다.

### 2.2.2 UserDaoTest의 특징
```java
package springboot.dao;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.GenericXmlApplicationContext;
import springboot.domain.User;

import java.sql.SQLException;

public class UserDaoTest {
    public static void main(String[] args) throws SQLException, ClassNotFoundException {
        //ConnectionMaker connectionMaker = new DConnectionMaker();
        //UserDao dao = new UserDao(connectionMaker);
        //UserDao dao = new DaoFactory().userDao(); // 팩토리를 사용함.
        //ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
        ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
        UserDao dao = context.getBean("userDao", UserDao.class);
        User user = new User();
        user.setId("wh225121");
        user.setName("백기선2");
        user.setPassword("married");

        dao.add(user);

        System.out.println(user.getId() + "등록 성공");

        User user2 = dao.get(user.getId());
        System.out.println(user2.getName());

        System.out.println(user2.getPassword());

        System.out.println(user.getId() + "조회 성공");

    }
}
```

#### 웹을 통한 DAO 테스트 방법의 문제점

- 웹 화면을 통해 값을 입력하고, 기능을 수행하고, 결과를 확인하는 방법은 가장 흔히 쓰이는 방법이지만, DAO에 대한 테스트로서는 단점이 너무 많다.
- 모든 레이어의 기능을 다 만들고 나서야 테스트가 가능하다.
- 다른 계층의 코드와 컴포넌트, 심지어 서버의 설정 상태까지 모두 테스트에 영향을 줄 수 있다. 이런 방식으로 테스트하는 것은 번거롭고, 오류가 있을 때 빠르고 정확하게 대응하기가 힘들다.

#### 작은 단위의 테스트

- 테스트는 가능하면 작은 단위로 쪼개서 집중해서 할 수 있어야 한다.

- 관심사의 분리라는 원리가 여기에도 적용된다. 테스트의 관심이 다르다면 테스트할 대상을 분리하고 집중해서 접근해야 한다.

- UserDaoTest처럼 작은 단위의 코드에 대해 테스트를 수행한 것을 단위 테스트(unit test)라고 한다.


- **DB가 사용되면 단위 테스트가 아닐까?**
    - 어떤 개발자는 테스트 중에 DB가 사용되면 단위 테스트가 아니라고도 한다. 그럼 UserDaoTest는 단위 테스트가 아니라고 봐야 할까? => 아니다
    - 지금까지 UserDaoTest를 수행할 때 매번 테이블의 내용을 비웠다.
    - 사용할 DB의 상태를 테스트가 관장하고 있다면 단위 테스트라고 해도 된다.
    - 다만, 통제할 수 없는 외부의 리소스에 의존하는 테스트는 단위 테스트가 아니라고 보기도 한다.
       - DB의 상태가 매번 달라지고, 테스트를 위해 DB를 세팅할 수 없다면? UserDaoTest가 단위 테스트로서 가치가 없어진다.

- **통합 테스트도 필요하다**
- 길고 많은 단위가 참여하는 테스트(통합테스트)도 필요하다.
- 각종 기능을 모두 사용한 다음에 로그아웃까지 하는 전 과정을 묶어 테스트할 필요가 있다.
- 각 단위 기능은 잘 동작하는데 묶어놓으면 안 되는 경우가 종종 발생하기 떄문이다.
