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

