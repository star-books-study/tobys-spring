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
- get()이 파라미터로 주어진 id에 해당하는 사용자를 가져온 것인지, 그냥 아무거나 가져온 것인지 테스트에서 검증하지는 못했다.
- User를 하나 더 추가해서 두 개의 User를 add()하고, 각 User의 id를 파라미터로 전달해서 get()을 실행하도록 만들자.

```java
@Test
public void addAndGet() throws SQLException {
	..
	UserDao dao = applicationContext.getBean("userDao", UserDao.class);
	
	User user1 = new User("gyumee", "박성철", "springno1");
	User user2 = new User("yel-m", "정예림", "springno2");

	dao.deleteAll();
	assertThat(dao.getCount(), is(0));

	dao.add(user1);
	dao.add(user2);
	assertThat(dao.getCount(), is(2));

	User userget1 = dao.get(user1.getId());
	assertThat(userget1.getName(), is(user1.getName());
	assertThat(userget1.getPassword(), is(user1.getPassword());
	
	User userget2 = dao.get(user1.getId());
	assertThat(userget2.getName(), is(user2.getName());
	assertThat(userget2.getPassword(), is(user2.getPassword());
}
```
#### get() 예외 조건에 대한 테스트
- get() 메소드에 전달된 id 값에 해당하는 사용자 정보가 없을 때
	- `null`과 같은 특별한 값을 리턴한다
	- id에 해당하는 정보를 찾을 수 없다고 예외를 던진다
- 각기 장단점이 있다. 여기서는 후자의 방법을 써보자.

- 테스트 진행 중에 특정 예외가 던져지면 테스트가 성공한 것이고, 예외가 던져지지 않고 정상적으로 작업을 마치면 테스트가 실패했다고 판단해야 한다.

- 문제는 예외 발생 여부는 메소드를 실행해서 리턴 값을 비교하는 방법으로 확인할 수 없다는 점이다. 이런 경우를 위해 JUnit은 특별한 방법을 제공해준다.

```java
@Test(expected=EmptyResultDataAccessException.class)
public void getUserFailure() throws SQLException {
	ApplicationContext context = new GenericXmlApplicationContext("applicationContext.xml");

	UserDao dao = context.getBean("userDao", UserDao.class);
	dao.deleteAll();
	assertThat(dao.getCount(), is(0));

	dao.get("unknown_id"); // 이 메서드 실행 중에 예외가 발생해야 한다. 예외가 발생하지 않으면 테스트가 실패한다.
}
```

- `@Test`에 `expected`를 추가해놓으면
	- 정상적으로 테스트 메소드를 마치면 테스트가 실패하고,
	- expected에서 지정한 예외가 던져지면 테스트가 성공한다.
- 예외가 반드시 발생해야 하는 경우를 테스트하고 싶을 때 유용하게 쓸수 있다.
- 그런데 이 테스트를 실행시키면 테스트는 실패한다.
  - `get()` 메서드에서 `rs.next()`를 실행시킬 때 가져올 로우가 없다는 SQLException이 발생할 것이다.


#### 테스트를 성공시키기 위한 코드의 수정

- 이제부터 할 일은 이 테스트가 성공하도록 get() 메소드 코드를 수정하는 것이다.

```java
public User get(String id) throws SQLException {
        ...
	ResultSet rs = ps.executeQuery();
        
        User user = null; // User는 null 상태로 초기화해놓는다.

	// id를 조건으로 한 쿼리의 결과가 있으면 User 오브젝트를 만들고 값을 넣어준다.
        if(rs.next()){  
            user = new User();
            user.setId(rs.getString("id"));
            user.setName(rs.getString("name"));
            user.setPassword(rs.getString("password"));
        }

        rs.close();
        ps.close();
        c.close();

	// 결과가 없으면 User는 null 상태 그대로일 것이다. 이를 확인해서 예외를 던져준다.
        **if(user == null) throw new EmptyResultDataAccessException(1);** 

        return user;
    }
```

#### 포괄적인 테스트
- 이렇게 DAO의 메서드에 대한 포괄적인 테스트를 만들어두는 편이 훨씬 안전하고 유용하다.
- 개발자가 테스트를 직접 만들 떄 자주 하는 실수가 성공하는 테스트만 골라서 만드는 것이다. 
- 스프링의 창시자인 로드 존슨은 항상 네거티브 테스트를 먼저 만들라는 조언을 했다.

- **테스트를 작성할 때 부정적인 케이스를 먼저 만드는 습관을 들이자!**

- get() 메소드의 경우

	- 존재하지 않는 id가 주어졌을 때는 어떻게 반응할지를 먼저 결정하고, 이를 확인할 수 있는 테스트를 먼저 만들어야 한다.

### 2.3.4 테스트가 이끄는 개발
- 많은 전문적인 개발자가 테스트를 먼저 만들어 테스트가 실패하는 것을 보고 나서 코딩하는 방법을 개발 방법을 적극적으로 사용하고 있다.

#### 기능설계를 위한 테스트
- getUserFailure() 테스트 코드의 내용을 정리해보면 다음과 같다.

|    | 단계   | 내용             | 코드                  |
-----|------|----------------|----------------------|
| 조건  | 어떤 조건을 가지고 | 가져올 사용자 정보가 존재하지 않는 경우에 | dao.deleteAl(); assertThat(dao.getCount(), is(0)); |
| 행위 | 무엇을 할 때 | 존재하지 않는 id로 get()을 실행하면 | get("unknown_id"); |
| 결과 | 어떤 결과가 나온다. | 특별한 예외가 던져진다 | @Test(expected=EmptyResultDataAccessException.calss) |


- 테스트 코드는 마치 잘 작성된 하나의 기능정의서처럼 보인다.
- 보통 기능설계, 구현, 테스트라는 일반적인 개발 흐름의 기능설계에 해당하는 부분을 테스트 코드가 일부분 담당하고 있다.

#### 테스트 주도 개발(TDD, Test Driven Development)

- 테스트 코드를 먼저 만들고, 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방식.
만들고자 하는 기능의 내용을 담고 있으면서(기능 설계) 만들어진 코드를 검증도 해줄 수 있다.
장점

- 테스트를 먼저 만들고 그 테스트가 성공하도록 하는 코드만 만드는 식으로 진행하기 때문에 테스트를 빼먹지 않고 꼼꼼하게 만들어낼 수 있다.
- 테스트를 작성하는 시간과 애플리케이션 코드를 작성하는 시간의 간격이 짧아진다. 이미 테스트를 만들어뒀기 때문에 코드를 작성하면 바로바로 테스트를 실행해볼 수 있기 때문이다. 그 덕분에 코드에 대한 피드백을 매우 빠르게 받을 수 있게 된다.
- 매번 테스트가 성공하는 것을 보면서 작성한 코드에 대한 확신을 가질 수 있어, 가벼운 마음으로 다음 단계로 넘어갈 수가 있다. 자신감과 마음의 여유가 생긴다.
- TDD에서는 테스트 작성하고 이를 성공시키는 코드를 만드는 작업의 주기를 가능한 한 짧게 가져가도록 권장한다.


- 과거 개발자들은 엔터프라이즈 애플리케이션의 테스트를 만들기가 매우 어렵다고 생각해 테스트 코드를 짜지 않았다.
- 하지만 스프링은 테스트하기 편리한 구조의 애플리케이션을 만들게 도와줄 뿐만 아니라, 엔터프라이즈 애플리케이션 테스트를 빠르고 쉽게 작성할 수 있는 매우 편리한 기능을 많이 제공하므로 테스트를 하자!

### 2.3.5 테스트 코드 개선
- 테스트 코드 자체가 이미 자신에 대한 테스트이기 때문에 테스트 결과가 일정하게 유지된다면 얼마든지 리팩토링을 해도 좋다.


#### @Before
- UserDaoTest에서 스프링의 애플리케이션 컨텍스트를 만드는 부분과 컨텍스트에서 UserDao를 가져오는 부분이 반복된다.
- 반복적으로 등장하는 앞의 코드를 제거하고 setUp()에 넣어주고, dao 변수를 테스트 메서드에서 접근할 수 있도록 인스턴스 변수로 변경한다 마지막으로 setUp() 메서드에 @Before이라는 애노테이션을 추가한다.
- @Before : Junit에서 제공하는 애노테이션. @Test 메서드가 실행되기 전에 먼저 실행돼야 하는 메서드를 정의한다.

```java
// 2-15. 중복된 코드를 제거한 UserDaoTest
import org.junit.Before;
...
public class UserDaoTest {
	private UserDao dao; // setUp() 메서드에서 만드는 오브젝트를 테스트 메서드에서 사용할 수 있도록 인스턴스 변수로 선언

	// 테스트 메서드에 반복적으로 나타났던 코드를 제거하고 별도의 메서드로 옮긴다.
	@Before 
	public void setUp() {
		ApplicationContext = new GenericXmlApplicationContext("applicationContext.xml");
		this.dao = context.getBean("userDao", userDao.class);
	}
	...

	@Test
	public void addAndGet() throws SQLException {
		...
	}

	@Test(expected=EmptyResultDataAccessException.class)
	public void addAndGet() throws SQLException {
		...
	}
}
```
- JUnit이 하나의 테스트 클래스를 가져와 테스트를 수행하는 방식은 다음과 같다.

	1. 테스트 클래스에서 @Test가 붙은 public이고 void형이며 파라미터가 없는 테스트 메소드를 모두 찾는다.
	2. 테스트 클래스의 오브젝트를 하나 만든다.
	3. @Before가 붙은 메소드가 있으면 실행한다.
	4. @Test가 붙은 메소드를 하나 호출하고 테스트 결과를 저장해둔다.
	5. @After가 붙은 메소드가 있으면 실행한다.
	6. 나머지 테스트 메소드에 대해 2~5번을 반복한다.
	7. 모든 테스트의 결과를 종합해서 돌려준다.
	  <img width="406" alt="스크린샷 2024-04-14 오후 12 05 45" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/e572c5af-583f-4fb2-b9c9-65e9ce19510f">

- 보통 하나의 테스트 클래스 안에 있는 테스트 메소드들은 공통적인 준비 작업과 정리 작업이 필요한 경우가 많다.
- 이런 작업들을 @Before, @After가 붙은 메소드에 넣어두면 JUnit이 자동으로 메소드를 실행해주니 매우 편리하다.
- 각 테스트 메소드에서 직접 setUp()과 같은 메소드를 호출할 필요도 없다.

- e대신 @Before나 @After 메소드를 테스트 메소드에서 직접 호출하지 않기 때문에 서로 주고받을 정보나 오브젝트가 있다면 인스턴스 변수를 이용해야 한다.
- UserDaoTest에서는 스프링 컨테이너에서 가져온 UserDao 오브젝트를 인스턴스 변수 dao에 저장해뒀다가, 각 테스트 메소드에서 사용하게 만들었다.
- 꼭 기억해야할 사항은 각 **테스트 메서드를 실행할 때마다 테스트 클래스의 오브젝트를 새로 만든다**는 점이다.
- 왜 테스트 메소드를 실행할 때마다 새로운 오브젝트를 만드는 것일까? 그냥 테스트 클래스마다 하나의 오브젝트만 만들어놓고 사용하는 편이 성능도 낫고 더 효율적이지 않을까?

	- JUnit 개발자는 각 테스트가 서로 영향을 주지 않고 독립적으로 실행됨을 확실히 보장해주기 위해 매번 새로운 오브젝트를 만들게 했다.
	- 덕분에 인스턴스 변수도 부담 없이 사용할 수 있다. 어차피 다음 테스트 메소드가 실행될 때는 새로운 오브젝트가 만들어져서 다 초기화될 것이다.
 
#### 픽스처

- 테스트를 수행하는 데 필요한 정보나 오브젝트를 fixture라고 한다.

- 픽스처는 여러 테스트에서 반복적으로 사용되기 때문에 `@Before` 메소드를 이용해 생성해두면 편리하다.
- UserDaoTest에서라면 dao가 대표적인 픽스처이다.
- add() 메서드에 전달하는 User 오브젝트들도 픽스처라고 볼 수 있다.
  - 이 부분에서 중복된 코드를 제거하기 위해 @Before 메서드로 추출해보자.
	```java
	@Before 
	public void setUp() {
		...
		this.user1 = new User("1233", "박성철", "springno1");
		this.user2 = new User("1111", "정예림", "springno2");
		this.user3 = new User("1e3334", "박범진", "springno3");
	}
 	```

## 2.4 스프링 테스트 적용
- 찜찜한 부분 : @Before 메서드가 테스트 메서드 개수만큼 반복 -> 어플리케이션 컨텍스트도 여러 번 생성
- 빈이 많아지고 복잡해지면 애플리케이션 컨텍스트 생성해 시간이 걸릴 수 있다.
- 테스트를 마칠 때마다 애플리케이션 컨텍스트 내의 빈이 할당한 리소스 등을 깔끔하게 정리해주지 않으면 다음 테스트에서 새로운 애플리케이션 컨텍스트가 만들어지면서 문제를 일으킬 수 있다.

- 테스트는 일관성있는 실행 결과를 보장해야 하고, 테스트의 실행 순서가 결과에 영향을 미치지 않아야 한다.
- 다행히도 애플리케이션 컨텍스트는 초기화되고 나면 내부의 상태가 바뀌는 일은 거의 없다. 왜냐하면 빈이 싱글톤으로 만들어졌기 때문에 상태를 갖지 않기 때문이다.

- UserDao 빈을 가져다가 add(), get()을 사용한다고 해서 UserDao 빈의 상태가 바뀌진 않는다. 
- 애플리케이션 컨텍스트는 특별한 경우가 아니면 여러 테스트가 공유해서 사용해도 된다.

- 여러 테스트가 함께 참조한 애플리케이션 컨텍스트를 오브젝트 레벨이 아닌 스태틱 필드에 저장해두면 어떨까?
- JUnit은 테스트 클래스 전체에 걸쳐 딱 한 번만 실행되는 `@BeforeClass` 스태틱 메서드를 지원한다.

### 2.4.1 테스트에서 애플리케이션 컨텍스트 관리
- 스프링은 JUnit을 이용하는 테스트 컨텍스트 프레임워크를 제공한다.
- 애플리케이션 컨텍스트를 만들어서 모든 테스트가 공유하게 할 수 있다.

#### 스프링 테스트 컨텍스트 프레임워크 적용

```java
@RunWith(SpringJUnit4ClassRunnder.class) // 스프링의 테스트 컨텍스트 프레임워크의 JUnit 확장 기능 지정
@ContextConfiguration(locations="/applicationContext.xml // 테스트 컨텍스트가 자동으로 만들어줄 애플리케이션 컨텍스트의 위치 지정
public class UserDaoTest {
	@Autowired ApplicationContext applicationContext; // 테스트 오브젝트가 만들어지고 나면 스프링 컨텍스트에 자동으로 값이 주입된다.
	UserDao userDao;
	
	@Before
	public void setUp() {
		this.userDao = this.applicationContext.getBean("userDao", UserDao.class);
		...
	}
```
- context 변수에 애플리케이션 컨텍스트가 들어있다. 스프링 컨텍스트 프레임워크의 JUnit 확장기능의 마법 🧙‍♀️

- `@RunWith`는 JUnit5에서 테스트 클래스를 확장할 때 쓰이는 애노테이션이다.
- `@ContextConfiguration`은 자동으로 만들어줄 애플리케이션 컨텍스트의 설정 파일 위치를 지정한 것이다. 
