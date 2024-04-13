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

- UserDaoTest는 UserDao라는 작은 단위의 데이터 액세스 기능만을 테스트하기 위해 만들어졌다.
- 이렇게 작은 단위의 코드에 대해 테스트를 수행한 것을 단위 테스트(unit test)라고 한다.


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
> 단위별로 테스트를 먼저 모두 진행하고 나서 이런 긴 테스트를 시작하면 이미 각 단위별로 충분한 검증을 마쳤으므로 훨씬 나을 것이다.

#### 자동수행 테스트 코드
- UserDaoTest는 테스트할 데이터가 코드를 통해 제공되고, 테스트 작업도 코드를 통해 자동으로 실행된다.
- 이렇게 테스트가 자동 수행되지 않는다면?
    - 간혹 테스트 값 입력을 실수가 있어서 오류가 나면 다시 테스트를 반복해야 한다.
    - 테스트를 위해 서버, 브라우저, 주소 입력 등 귀찮은 작업이 필요하다.
- 테스트는 자동으로 수행되도록 만들어지는 것이 중요하다.
  - 자동으로 수행되는 테스트는 자주 반복할 수 있다.
  - 번거로운 작업 없이 테스트를 빠르게 수행할 수 있다.

- **테스트용 클래스는 분리해야 한다**
  - 애플리케이션을 구성하는 클래스 안에 테스트 코드를 포함시키는 것보다는 별도로 테스트용 클래스를 만들어서 테스트 코드를 넣는 편이 낫다.


#### 지속적인 개선과 점진적인 개발을 위한 테스트
- 우리는 일단 단순 무식한 방법으로 정상작동하는 코드를 만들고, 테스트를 만들어뒀기 때문에 매우 작은 단계를 거쳐가며 계속 코드를 개선할 수 있었다.
- 테스트를 이용하면 새로운 기능이 정상작동하는지 확인할 수 있을 뿐만 아니라 기존에 만들어뒀던 기능들이 영향을 받지 않고 여전히 잘 동작하는지를 확인할 수도 있다.

### 2.1.3 UserDaoTest의 문제점
1. **수동 확인 작업의 번거로움**

- UserDaoTest는 테스트를 수행하는 과정과 입력 데이터의 준비를 모두 자동으로 진행하도록 만들어졌다.

- 하지만 여전히 사람의 눈으로 확인하는 과정이 필요하다.
- `add()`에서 User 정보를 DB에 등록하고, 이를 다시 `get()`을 이용해 가져왔을 때 **입력한 값과 가져온 값이 일치하는지**를 테스트 코드는 확인해주지 않는다.


2. **실행 작업의 번거로움**

- 아무리 간단히 실행 가능한 main() 메소드라고 하더라도 매번 그것을 실행하는 것은 제법 번거롭다.
- 만약 DAO가 수백 개가 되고 그에 대한 main() 메소드도 그만큼 만들어진다면?

    - 전체 기능을 테스트해보기 위해 main() 메소드를 수백 번 실행해야 한다.
## 2.2 UserDaoTest 개선
### 2.2.1 테스트 검증의 자동화

- 모든 테스트는 성공과 실패의 두 가지 결과를 가질 수 있다.

    - 테스트 에러 : 테스트가 진행되는 동안에 에러가 발생해서 실패 -> 콘솔에 뜨는 에러 메시지와 호출 스택 정보를 통해 확인 가능
    - 테스트 실패 : 테스트 작업 중에 에러가 발생하진 않았지만 그 결과가 기대한 것과 다르게 나옴 -> 별도의 확인 작업이 필요

```java
// 2-2. 수정 전 테스트 코드
public static void main(String[] args) throws SQLException, ClassNotFoundException {
    ...
    
    System.out.println(user2.getName());
    System.out.println(user2.getPassword());
    System.out.println(user2.getId() + " 조회 성공");
}
```

```
// 2-3. 수정 후 테스트 코드
public static void main(String[] args) throws SQLException, ClassNotFoundException {
    ...
    
    if (!user.getName().equals(user2.getName())) {
        System.out.println("테스트 실패 (name)");
    } else if (!user.getPassword().equals(user2.getPassword())) {
        System.out.println("테스트 실패 (password)");
    } else {
        System.out.println("조회 테스트 성공");
    }
}
```
- 직접 출력 값을 하나씩 확인 안 해도 테스트 성공이란 출력으로 무사히 진행됨을 알 수 있다. 실패했더라도 어디서 실패하는 지 출력한다.

- 자동화된 테스트를 위한 xUnit 프레임워크를 만든 켄트 벡은 "테스트란 개발자가 마음 편하게 잠자리에 들 수 있게 해주는 것"이라 했다.


### 2.2.2 테스트의 효율적인 수행과 결과 관리
- main() 메서드로 만든 테스트 작성 방법으로는 애플리케이션 규모가 커지고 테스트 개수가 많아지면 테스트 수행이 점점 부담이 될 것
 => `JUnit` : 프로그래머를 위한 자바 테스팅 프레임워크. 자바로 단위 테스트 만들 때 유용하게 사용

#### JUnit 테스트로 전환
- 이제 테스트용 main() 메소드를 Junit으로 전환해보자.
- JUint 역시 제어의 권한을 담당하는 프레임워크
  -> 개발자가 만든 클래스의 오브젝트를 생성하고 실행하는 일은 프레임워크에 의해 진행
  -> main() 메서드를 만들 필요도, 오브젝트를 만들어서 실행시키는 코드를 만들 필요도 필요 없음

#### 테스트 메서드 전환
- main() 메서드 테스트는 프레임워크에 적용하기에 적합하지 않음.
  - main() 메서드로 만들어졌다는 건 제어권을 직접 갖는다는 의미
- 새로 만들 테스트 메소드는 JUnit 프레임워크가 요구하는 조건 두가지를 따라야 한다.
    1. 메소드가 `public`으로 선언
    2. 메소드에 `@Test` 어노테이션을 붙임

```java
// 2-4. JUnit 프레임워크에서 동작할 수 있는 테스트 메서드로 전환
import org.junit.Test;
...
public class UserDaoTest {

    @Test
    public void addAndGet() throws SQLException {
        ApplicationContext context = new
            ClassPathXmlApplicationContext("applicationContext.xml");

        UserDao dao = context.getBean("userDao", UserDao.class);
        ...
    }
}
```
- 일반 메서드로 만들고 적절한 이름을 붙여준다.

#### 검증 코드 전환
- if/else 문장을 JUnit이 제공하는 assertThat()이라는 스태틱 메소드를 이용해 전환한다.
```
asserThat(user2.getName(), is(user.getName()));
```

- assertThat 메소드의 첫 번째 파라미터의 값을 두번째 파라미터로 준 매처(조건)과 비교해서
일치하면 다음으로 넘어가고, 아니면 테스트가 실패하도록 만듭니다.
- `is()` 매처는 equals()로 비교해주는 기능을 가졌다.


```java
// 2-5. JUnit을 적용한 UserDaoTest
import static org.hamcrestCoreMatchers.is;
import static org.junit.Assert.assertThat;
...
public class UserDaoTest {
	
	@Test 
	public void andAndGet() throws SQLException {
		ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");
		UserDao dao = context.getBean("userDao", UserDao.class);
		
		User user = new User();
		user.setId("hunny");
		user.setName("hunny");
		user.setPassword("spring");

		dao.add(user);
			
		User user2 = dao.get(user.getId());
		
		assertThat(user2.getName(), is(user.getName()));
		assertThat(user2.getPassword(), is(user.getPassword()));
	}
}
```

#### JUnit 테스트 실행
- JUnit 프레임워크도 어디선가 한 번은 JUnit 프레임워크를 시작시켜 줘야 한다.
- 어디에든 main() 메서드를 하나 추가하고, 그 안에 JUnitCore 클래스의 main 메서드를 호출해주는 간단한 코드를 넣어주면 된다.
- 메서드 파라미터에는 @Test 테스트 메서드를 가진 클래스의 이름을 넣어준다.

```java
// 2-6. JUnit을 이용해 테스트를 실행해주는 main() 메서드
import org.junit.runner.JUnitCore;
...
public static void main(String[] args) {
	jUnitCore.main("spring.user.dao.UserDaoTest");
}
```
- 테스트 에러 : JUnit은 assertThat()을 이용해 검증을 했을 때 기대한 결과가 아니면 이 `AssertionError`를 던진다. 따라서 assertThat()의 조건을 만족하지 못하면 테스트는 더 이상 진행되지 않고 JUnit은 테스트가 실패했음을 알게 된다.
- 테스트 예외 : 테스트 수행 중에 일반 예외가 발생한 경우에도 마찬가지로 테스트 수행은 중단되고 테스트는 실패한다.

## 2.3 개발자를 위한 테스팅 프레임워크 JUnit
### 2.3.1 JUnit 테스트 실행 방법
- 가장 좋은 JUnit 테스트 실행 방법은 자바 IDE에 내장된 JUnit 테스트 지원 도구를 사용하는 것이다.

#### IDE

- IDE를 통해 JUnit 테스트의 실행과 그 결과를 확인하는 방법
- 매우 간단하고 직관적이며 소스와 긴밀하게 연동돼서 결과를 볼 수 있다.

#### 빌드 툴

- 여러 개발자가 만든 코드를 모두 통합해서 테스트를 수행해야 할 때도 있다.

- 이런 경우에는 서버에서 모든 코드를 가져와 통합하고 빌드한 뒤에 테스트를 수행하는 것이 좋다.
- 이때는 빌드 스크립트를 이용해 JUnit 테스트를 실행하고 그 결과를 메일 등으로 통보받는 방법을 사용하면 된다.

### 2.3.2 테스트 결과의 일관성
- 테스트가 외부 상태에 따라 성공하기도 하고 실패하기도 하면 안된다.
- 일관성있는 결과를 위해 DB 초기화가 필요하다.
- UserDaoTest의 문제는 이전 테스트 떄문에 DB에 등록된 중복 데이터가 있을 수 있다는 점이다.
- 가장 좋은 해결책은 테스트를 마치고 나면 등록한 사용자 정보를 삭제하는 것이다.

#### deleteAll()의 getCount() 추가

- deleteAll 추가
	```java
 	// 2-7. deleteAll 메서드
	public void deleteAll() throws SQLException {
		Connection c = dataSource.getConnection();
		PreparedStatement ps = c.prepareStatement("delete from users");
		
		ps.executeUpdate();
		
		ps.close();
		c.close();
	}
	```
- getCount()
  - USER 테이블의 레코드 개수를 돌려준다.
  ```java
  // 2-8. getCount() 메서드
	public int getCount() throws SQLException {
		Connection c = dataSource.getConnection();
		PreparedStatement ps = c.prepareStatement("select count(*) from users");
		
		ResultSet rs = ps.executeQuery();
		rs.next();
		int count = rs.getInt(1);
		
		rs.close();
		ps.close();
		c.close();
		
		return count;
	}
  ```

#### deleteAll과 getCount()의 테스트
- deleteAll()과 getCount()는 자동화돼서 반복적으로 실행 가능한 테스트 방법은 아니다.
- deleteAll()을 테스트가 시작될 떄 실행해보자.
- deleteAll()을 검증하기 위해 getCount()를 이용하자. 기대한대로 작동한다면 레코드의 개수가 0이 나와야 한다.
- getCount()를 검증하기 위해서는 add() 메서드 실행 후 레코드의 개수가 0에서 1이 되었는지 확인해보자.

```java
// 2-9. deleteAll()과 getCount()가 추가된 addAndGet() 테스트
@Test
public void addAndGet() throws SQLException {
	ApplicationContext applicationContext = new GenericXmlApplicationContext("spring/applicationContext.xml");
	UserDao userDao = applicationContext.getBean(UserDao.class);
	
	// deleteAll(), getCount() 기능 동작 확인
	userDao.deleteAll();
	assertEquals(userDao.getCount(), 0);
	
	User userToAdd = new User();
	userToAdd.setId("jinkyu1");
	userToAdd.setName("진규");
	userToAdd.setPassword("password");
	userDao.add(userToAdd);

	// 유저가 있을 때, getCount() 기능 동작 확인
	assertEquals(userDao.getCount(), 1);
	
	User userToGet = userDao.get("jinkyu1");

	// 유저가 제대로 등록되었는지 확인
	assertEquals(userToAdd.getId(), userToGet.getId());
	assertEquals(userToAdd.getName(), userToGet.getName());
	assertEquals(userToAdd.getPassword(), userToGet.getPassword());
	
	// 유저가 있을 때, deleteAll(), getCount() 기능 동작 확인
	userDao.deleteAll();
	assertEquals(userDao.getCount(), 0);
}
```

#### 동일한 결과를 보장하는 테스트
- add()를 호출해 데이터를 넣었으므로 검증이 끝나면 deleteAll()을 실행해서 등록된 레코드를 삭제해주는 방법도 생각해볼 수 있다.
- 그러나 이전에 어떤 작업을 하다가 테스트를 실행할지 알 수 없다.
- 그보다는 테스트하기 전에 테스트 실행에 문제가 되지 않는 상태를 만들어주는 편이 더 낫다.
- 테스트는 항상 일관성 있는 결과가 보장되어야 한다.


### 2.3.3 포괄적인 테스트
- 테스트를 안 만드는 것도 위험한 일이지만, 성의 없이 테스트를 만드는 바람에 문제가 있는 코드인데도 테스트가 성공하게 만드는 건 더 위험하다.

#### getCount() 테스트
- getCount()에 대한 좀 더 꼼꼼한 테스트를 만들어보자.

- 기존 테스트에서는 deleteAll()을 실행했을 때 테이블이 비어 있는 경우(0)와 add()를 한 번 호출한 뒤의 결과(1)뿐이다.
- 두 개 이상의 레코드를 add() 했을 때는 getCount()의 실행 결과가 어떻게 될까?
```java
@Test
public void count() throws SQLException {
	ApplicationContext contex = new GenericXmlApplicationContext("applicationContext.xml");

	UserDao dao = context.getBean("userDao", UserDao.class);
	User user1 = new User("user1", "박성철", "springno1");
	User user2 = new User("user2", "이길원", "springno2");
	User user3 = new User("user3", "박범진", "springno3");

	dao.deleteAll(); // USER 테이블의 데이터 모두 지우기
	assertEquals(dao.getCount(), is(0)); // 레코드 개수가 0임을 확인
	
	dao.add(user1);
	assertEquals(dao.getCount(), is(1));
	
	userDao.add(user2);
	assertEquals(dao.getCount(), is(2));
	
	userDao.add(user3);
	assertEquals(dao.getCount(), is(3));

}
```
- 테스트는 순서에 영향받아선 안된다.
- 두 개의 테스트가 어떤 순서로 실행될지는 알 수 없다는 것이다. (JUnit은 특정한 테스트 메소드의 실행 순서를 보장해주지 않는다.)
- 테스트의 결과가 테스트 실행 순서에 영향을 받는다면 테스트를 잘못 만든 것이다.
#### addAndGet() 테스트 보완
```java
@Test
    @DisplayName("회원 추가 및 불러오기")
    public void addAndGet() throws SQLException {
        ApplicationContext applicationContext = new GenericXmlApplicationContext("spring/applicationContext.xml");
        UserDao userDao = applicationContext.getBean(UserDao.class);

        // `deleteAll()`, `getCount()` 기능 동작 확인
        userDao.deleteAll();
        assertEquals(userDao.getCount(), 0);

        User user1 = new User();
        user1.setId("jinkyu1");
        user1.setName("진규");
        user1.setPassword("password");
        userDao.add(user1);
        // 유저가 있을 때, `getCount()` 기능 동작 확인
        assertEquals(userDao.getCount(), 1);

        User user2 = new User();
        user2.setId("jake2");
        user2.setName("제이크");
        user2.setPassword("password");
        userDao.add(user2);
        // 유저가 있을 때, `getCount()` 기능 동작 확인 2
        assertEquals(userDao.getCount(), 2);

        User user1Get = userDao.get("jinkyu1");
        // 유저가 제대로 불러와지는지 확인
        assertEquals(user1.getId(), user1Get.getId());
        assertEquals(user1.getName(), user1Get.getName());
        assertEquals(user1.getPassword(), user1Get.getPassword());

        User user2Get = userDao.get("jake2");
        // 항상 같은 유저를 불러오는 것은 아닌지, 유저가 제대로 불러와지는지 확인
        assertEquals(user2.getId(), user2Get.getId());
        assertEquals(user2.getName(), user2Get.getName());
        assertEquals(user2.getPassword(), user2Get.getPassword());

        // 유저가 있을 때, `deleteAll()`, `getCount()` 기능 동작 확인
        userDao.deleteAll();
        assertEquals(userDao.getCount(), 0);
    }
```
- id를 조건으로 해서 사용자를 검색하는 기능을 가진 get()에 대한 테스트는 조금 부족한 감이 있다.

- get()이 파라미터로 주어진 id에 해당하는 사용자를 가져온 것인지, 그냥 아무거나 가져온 것인지 테스트에서 검증하지는 못했다.
