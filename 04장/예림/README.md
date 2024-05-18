# 4장. 예외
- 예외와 관련된 코드는 자주 엉망이 되거나 무성의하게 만들어지기 쉽다.
- 때론 잘못된 예외처리 코드 때문에 찾기 힘든 버그를 낳을 수도 있고, 생각지 않았던 예외상황이 발생했을 때 상상 이상으로 난처해질 수도 있다.

## 4.1.사라진 SQLException
```java
public void deleteAll() throws SQLException {
  this.jdbcContext...
}
public void deleteAll() {
  this.jdbcTemplate...
}
```
- jdbcTemplate을 사용할 시 throws SQLException 가 사라졌다. 어디로 갔을까???

- `SQLException`은 JDBC API 메소드들이 던져주는 것이므로 당연히 있어야 한다.
- 어째서 내부적으로 JDBC API를 쓰는 `jdbcTemplate`가 SQLException을 바깥으로 던지지 않는가?
### 4.1.1 초난감 예외 처리
- 왜 SQLException이 사라졌는지 알아보기 전에 초난감 예외처리들의 대표 선수🤾들을 살펴보자
#### 예외 블랙홀
```java
// 초난감 예외처리 코드
try {
  ...
} catch (Exception e) {}
```
- 예외를 잡고는 아무 것도 하지 않으면 위험하다.
- 위험한 이유 : 프로그램 실행 중에 어디선가 오류가 있어서 예외가 발생했는데 그것을 무시하고 계속 진행해버리기 때문이다.
- 어떤 기능이 비정상적으로 동작하거나, 메모리나 리소스가 소진될 수 있다!

```java
// 초난감 예외처리 코드2
} catch (SQLException e) {
  System.out.println(e);
}
```
```java
// 초난감 예외처리 코드3
} catch (SQLException e) {
  e.printStackTrace();
}
```
- 콘솔 로그를 누군가 계속 모니터링하지 않는 이상 위험하다
- catch 블록을 이용해 화면에 메시지를 출력한 것은 예외를 처리한 게 아니다.

```java
// 그나마 나은 예외처리
} catch (SQLException e) {
  e.printStackTrace();
  System.exit(1);
}
```
- 차라리 시스템을 종료해라
- 굳이 예외를 잡아서 조치를 취할 방법이 없다면 잡지 말아야 한다.
- 메서드에 throws SQLException을 선언해서 예외 처리 책임을 전가해버려라.

#### 무의미, 무책임한 throws
```java
// 초난감 예외처리4
public void method1() throws Exception {
  method2();
  ...
}

public void method2() throws Exception {
  method3();
  ...
}

public void method3() throws Exception ...
```
- 메소드 선언에 throws Exception을 기계적으로 붙였다
- catch 블록으로 예외를 잡아봐야 해결할 방법도 없고 JDK API나 라이브러리가 던지는 각종 이름도 긴 예외들을 처리하는 코드를 매번 throws로 선언하기도 귀찮아진 개발자의 임시방편
- 메소드 선언에서 의미 있는 정보를 얻을 수 없다. (정말 무엇인가 실행 중에 예외적인 상황이 발생할 수 있다는 것인지, 아니면 그냥 습관적으로 복사해서 붙여놓은 것인지 알 수가 없다.)
- 결국 이런 메소드를 사용하는 메소드 역시 throws Exception을 따라서 붙여야만 한다…!
- 결과적으로 적절한 처리를 통해 복구될 수 있는 예외상황도 제대로 다룰 수 있는 기회를 박탈당한다.
- 예외 처리에 대한 나쁜 습관은 어떤 경우에도 용납하지 않아야 한다.

### 1-2. 예외의 종류와 특징

- java.lang.Error 클래스의 서브 클래스
- 시스템에 비정상적 상황이 발생했을 경우 JVM단에서 발생한다.
- 어플리케이션에서 대응할 필요 없다!
#### Exception

java.lang.Exception 클래스와 그 서브 클래스로 정의되는 예외들
어플리케이션 코드 작업 중 예외상황이 발생했을 경우 사용된다.
