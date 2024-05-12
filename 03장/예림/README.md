# 3장. 템플릿

- 개방폐쇄원칙은 코드에서 어떤 부분은 변경을 통해 그 기능이 다양해지고 확장하려는 성질이 있고, 어떤 부분은 고정되어 있고 변하지 않으려는 성질이 있음을 말해준다.
- 템플릿 : 이렇게 변하는 성질이 다른 코드 중에서 변경이 거의 일어나지 않으며 일정한 패턴으로 유지되는 특성을 가진 부분을 자유롭게 변경되는 성질을 가진 부분으로부터 독립시켜서 효과적으로 활용할 수 있도록 하는 방법
        => 변하지 않는 특성을 변하는 특성을 가진 부분으로부터 독립시킨다!

## 3.1 다시보는 초난감 DAO
UserDao에는 예외 상황 처리에 관해서 심각한 문제점이 있다.

### 3.1.1 예외처리 기능을 갖춘 DAO


#### JDBC 수정 기능의 예외처리 코드

- 1장에서 만든 UserDao에 예외 처리를 추가한다. DB 커넥션 관련 코드는 리소스를 공유하기 때문에 예외가 발생하여 정상적으로 close()가 안 되면, 리소스 반환이 안 되어 나중에 리소스 부족과 관련된 에러가 발생할 수 있다. 일반적으로 서버에서는 제한된 DB 커넥션을 만들어 풀(Pool)로 관리한다.
- close()를 안 하면 다시 풀에 리소스를 넣지 못해서 반환하지 못한 커넥션이 쌓이면 다음 커넥션 요청 때 풀에 리소스가 없어서 서버 중단이 될 수 있다.
- 다음과 같이 try/catch/finally로 예외처리를 한다.

```java
 public void deleteAll() throws SQLException {
        Connection c = null;
        PreparedStatement ps = null;
        
        try {
            c = dataSource.getConnection();
            ps = c.prepareStatement("delete from users");
            ps.executeUpdate();
        } catch(SQLException e) {
        	throw e;
        } finally {
            if(ps!=null) {try {ps.close();} catch(SQLException e) {}}
            if(c!=null) {try{c.close();}catch(SQLException e) {}}
        }
    }
```

- deletaAll()에 예외처리를 적용했다. c나 ps가 null인 상태에서 close()를 호출하면 NullPointerException이 발생하기 때문에 위와 같이 null 여부를 우선 체크한 후 null이 아니면 close()를 호출하였다. close() 자체도 exception이 발생할 수 있기 때문에 try/catch문을 한 번 더 적용했다.

#### JDBC 조회 기능의 처리
- ResultSet도 반환해야 하는 리소스이기 때문에 close()가 필요
```java
public int getCount() throws SQLException {
        Connection c = null;
        PreparedStatement ps = null;
        ResultSet rs = null;

        try {
                c = dataSource.getConnection();
                ps = c.preparedStatement("select count(*) from users");

                rs = ps.executeQuery();
                rs.next();
                return rs.getInt(1);

        } catch (SQLException e) {
                throw e;
        } finally {
                if(rs != null) {
                        try {
                                rs.close();
                        } catch(SQLException e) {
                        }
                }
                ... 생략 ...
                // ps와 c를 순서대로 닫아준다. close()는 만들어진 순서의 반대로 하는 것이 원칙이다
```



## 3.2 변하는 것과 변하지 않는 것

### 3.2.1 JDBC try/catch/finally 코드의 문제점
- try/catch/finally가 계속해서 모든 메소드마다 반복된다. 중복의 문제가 발생하는 것이다.
 

### 3.2.2 분리와 재사용을 위한 디자인 패턴 적용

> UserDao 메서드를 개선해보자!

- 개선할 deleteAll() 메서드는 다음과 같다.
```java
Connection c =null;
PreparedStatement ps =null;
try (
        c = dataSource.getConnection();

        ps = c.prepareStatement("delete from users"); // 변하는 부분

        pS.executeUpdate();
} catch (5QLException e) {
        throw e;
} finally {
        if (ps != null) { try ( pS.close(); } catch (SQLException e) {} }
        if (c != null) { try (c.close(); } catch (5QLException e) {} }
}
```

#### 메서드 추출
- 메소드마다 `preparedStatement`로 SQL을 작성하는 부분을 제외하고 대부분 같은 코드가 반복된다.
- 이 코드를 분리해야한다. 우선 메소드 추출기법을 생각해볼 수 있겠는데 이 방법은 변하는 부분을 `makeStatement()`로 빼기 때문에 문제가 있다.
- 왜냐하면 보통 메소드 추출 기법은 공통된 코드를 빼서 여러 곳에서 재사용하기 위함인데, 이는 반대로 변하는 부분을 뺐기 때문에 재사용이 어렵다.
 

#### 템플릿 메소드 패턴 적용하기

- 템플릿 메소드 패턴은 상속을 통해 기능을 확장해서 사용하는 부분이다.
- 변하지 않는 부분은 슈퍼 클래스에 두고 변하는 부분은 추상 메서드로 정의 -> 서브 클래스에서 오버라이드
- makeStatement() 메서드를 다음과 같이 추상 메서드 선언으로 변경한다. (UserDao 클래스도 추상 클래스로 변경)
        ```java
        abstract protected PreparedStatement makeStatement(Connection c) throws SQLException;
        ```
- 그리고 이를 상속하는 서브 클래스를 만들어서 거기서 이 메서드 구현

```java
public class UserDaoDeleteAll extends UserDao {
    protected PreparedStatment makeStatement(Connection c) throws SQLException {
        PreparedStatement ps = c.prepareStatement("delete from users");
        return ps;
    }
}
```
- 그러나 이런식으로 사용하면 각각의 **DAO 메소드마다 클래스를 만들어야 한다**는 문제점이 있다.
- 또한 확장구조가 클래스 설계 시점에 고정되어 OCP원칙에 위배된다.

         <img width="575" alt="스크린샷 2024-04-30 오후 8 38 49" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/d23aa863-e34d-4d73-8b97-749bda3d938e">

=> 상속을 통해 확장을 꾀하는 템플릿 메서드 패턴 OUT



#### 전략 패턴의 적용
- 오브젝트를 아예 둘로 분리하고 클래스 레벨에서는 인터페이스를 통해서만 의존하도록 만드는 **전략 패턴**을 사용해보자 (OCP 준수 / 템플릿 메서드 패턴보다 유연)
- 전략 패턴은 확장에 해당하는 변하는 부분을 별도의 클래스로 만들어 추상화된 인터페이스를 통해 위임하는 방식
<img width="538" alt="image" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/b849231c-18c1-474f-a091-203d64e726df">

- Context의 contextMethod()에서 일정한 구조를 가지고 동작하다가 확장 기능은 Strategy 인터페이스를 통해 외부의 독립된 전략 클래스에 위임
- 우선 preparedStatement를 만드는 전략을 인터페이스(StatemStrategy.java)로 다음과 같이 만든다.

```java
public interface StatementStrategy {
    PreparedStatment makePreparedStatement(Connection c) throws SQLException;
}
```

- 그리고 이것을 구현한 전략(DeleteAllStatement.java)으로 만든다.

```java
public class DeleteAllStatement implements StatementStrategy{
    public PreparedStatement makePreparedStatemnt(Connection c) throws SQLException{
        PreparedStatement ps = c.preparedStatement("delete from users");
        return ps;
    }
}
```
 
- 이제 UserDao의 deleteAll() 메소드에서 이 전략 인스턴스를 생성한 후 사용하면 된다. 다음과 같다.


```java
public void deleteAll() throws SQLException{
    //생략
    try { 
        c = dataSource.getConnection();
        StatementStrategy strategy = new DeleteAllStatement();
        ps = strategy.makePreparedStatement(c);
    
        ps.executeUpdate();
    }catch(SQLException e){
        //생략
    }
}
```
- 전략 패턴의 방식을 적용했으나 문제점이 있다. deleteAll()이 컨텍스트의 역할을 하는데 컨텍스트가 구체적인 전략클래스인 DeleteAllStatement를 사용하도록 고정되어 있다.
- 만약 DeleteAllStatement가 수정되면 컨텍스트에 직접적인 영향을 미치기 때문에 OCP원칙에 잘 맞는 코드라고 볼 수 없다.
- 실제로 구체적인 전략클래스를 생성 및 사용하는 것은 컨텍스트의 역할이 아니라 '클라이언트'의 역할이기 때문에 컨텍스트와 클라이언트의 분리작업이 필요하다.

 
#### DI적용을 위한 클라이언트와 컨텍스트 분리

- 컨텍스트가 어떤 전략을 사용할지에 대해서는 **클라이언트가 결정**하도록 해야한다.
	- 클라이언트가 구체적인 전략의 하나를 선택하고 오브젝트로 만들어서 Context에 전달 -> Context는 전달받은 Strategy 구현 클래스의 오브젝트 사용
<img width="603" alt="image" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/4dbcee18-53d7-43b4-b67a-dd360c7bae3b">

- 위의 그림은 UserDao의 ConnectionMaker DI 방식이 따랐던 흐름과 유사하다. add()메소드가 필요한 커넥션 객체를 UserDaoTest 클라이언트에서 전략적으로 생성하여 UserDao에 넘겨준 후 add() 메소드가 이를 사용했다. 지금 상황에서도 이런 흐름에 맞게 개선할 필요가 있다.

- 여기서는 try/catch/finally 코드가 유사하게 반복되기 때문에 이 코드를 따로 빼서 컨텍스트로 만든다. 그리고 기존의 deleteAll()은 클라이언트 역할을 하도록 DeleteAllStatement()를 만든 후 컨텍스트의 인자로 넘겨 컨텍스트를 호출하는 역할로 변경한다. 다음과 같다.

```java
public void jdbcContextWithStatementStrategy(StatementStrategy stmt) throws SQLException{
    Connection c = null;
    PreparedStatement ps = null;
         
    try {
        c = dataSource.getConnection();
        ps = stmt.makePreparedStatement(c); // 주목
        ps.executeUpdate();
    } catch(SQLException e) {
        throw e;
    } finally {
        if(ps!=null) {try {ps.close();} catch(SQLException e) {}}
        if(c!=null) {try {ps.close();} catch(SQLException e) {}}
    }
}
```
- StatementStrategy stmt : 클라이언트가 컨텍스트를 호출할 때 넘겨줄 전략 파라미터
```java
// 클라이언트 역할
public void deleteAll() throws SQLException {
    StatementStrategy st = new DeleteAllStatement(); // 선정한 전략 클래스의 오브젝트 생성
    jdbcContextWithStatementStrategy(st);   // 컨텍스트 호출. 전략 오브젝트 전달
}
```
- 이제 클라이언트와 컨텍스트가 DI를 이용하여 분리된 코드로 개선되었다.

## 3.3 JDBC 전략 패턴의 최적화
- 

### 3.4 컨텍스트와 DI

#### 3.4.1 컨텍스트를 JdbcContext 클래스로 분리하기
- `jdbcContextWithStateStrategy()`는 JDBC의 기본적 흐름을 담고 있는 컨텍스트로, 다른 DAO에서도 사용 가능하다.
- 그렇기 때문에 UserDao에서 따로 클래스로 분리하는 작업을 해본다.

#### 클래스로 분리하기

- 일단 UserDao에 있는 컨테스트 코드를 `JdbcContext`라는 클래스를 만든 후에 `workWithStatementStrategy()` 라는 이름으로 변경하여 작성한다. 그리고 DataSource에 의존하기 때문에 DataSource도 setter로 DI받을 수 있도록 만든다. 
```java
public class JdbcContext {
    private DataSource dataSource;
	
    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }
	
    // 전략패턴 컨텍스트 역할
    public void workWithStatementStrategy(StatementStrategy stmt) throws SQLException{
    	 Connection c = null;
         PreparedStatement ps = null;
         
         try {
             c = dataSource.getConnection();
             ps = stmt.makePreparedStatement(c);
             ps.executeUpdate();
         } catch(SQLException e) {
         	throw e;
         } finally {
             if(ps!=null) {try {ps.close();} catch(SQLException e) {}}
             if(c!=null) {try {ps.close();} catch(SQLException e) {}}
         }
    }
}
```
- 이제 UserDao의 코드를 수정한다. 기존의 컨텍스트는 삭제한 후, JdbcContext를 DI받도록 만든다. 기존의 DataSource 코드는 아직 사용하고 있는 메소드들이 있기 때문에 그대로 냅둔다.

## 3.5 템플릿과 콜백
### 3.5.1 템플릿/콜백 동작원리
#### 템플릿/콜백 특징
- 전략패턴과 달리 콜백은 단일 메소드 인터페이스로 사용한다. 템플릿 작업 흐름 중 보통 한 번만 호출되기 때문이다.
- 즉 콜백은 **하나의 메소드를 가진 인터페이스를 구현한 익명 내부 클래스**라고 볼 수 있다.
- 또한 콜백 인터페이스 메소드는 보통 **파라미터**가 있는데, **템플릿 작업 흐름 중에 만들어지는 컨텍스트 정보를 전달 받을 때 사용**된다. `JdbcContext`의 `workWithStatementStrategy()` 메소드 내부에서 생성된 Connection 오브젝트가 콜백 메소드 `makePreparedStatement()` 파라미터로 넘어간다.

### 3.5.2 편리한 콜백의 재활용
#### 콜백의 분리와 재활용

- 콜백으로 전달하는 익명 내부 클래스의 코드를 보면 SQL 문장을 제외하고는 비슷한 코드가 반복된다.
- 콜백의 중복코드를 메소드 추출 방식으로 따로 빼낸 후 SQL 문장만 인자로 넘겨주도록 수정한다.

```java
public void deleteAll() throws SQLException {
	executeSql("delete from users");
}
    
private void executeSql(final String query) throws  SQLException {
	this.jdbcContext.workWithStatementStrategy(
		new StatementStrategy() {
	        	public PreparedStatement makePreparedStatement(Connection c)
				throws SQLException {
	            	return c.prepareStatement(query);
			}
		}
	);
}
```
- 바뀌지 않는 모든 부분을 빼내서 executeSql() 메서드로 만들었다.
- 바뀌는 부분인 SQL 문장만 파라미터로 받아서 사용하게 만들었다.

#### 콜백과 템플릿의 결합
- 재사용 가능한 콜백을 담고 있는 메서드라면 모든 DAO 메서드에서 executeSql() 메서드를 사용할 수 있도록 공유할 수 있는 템플릿 클래스 안으로 옮겨도 된다.
- executeSql 메서드를 JdbcContext로 옮긴다.

```java
public class JdbcContext {
	...
	public void executeSql(final String query) throws SQLException {
		workWithStatementStrategy(
			new StatementStrategy(
				public PreparedStatement makePreparedStatement(Connection c)
					throws SQLExceptioin {
				    return c.prepareStatement(query);
				}
			}
		);
	}
}
```
<img width="529" alt="스크린샷 2024-05-09 오후 9 59 40" src="https://github.com/star-books-coffee/tobys-spring/assets/101961939/d88b8d03-8db1-4ed1-899d-280512cba1ac">


### 3.5.3 템플릿 / 콜백의 응용
> 스프링에 내장된 것을 원리도 알지 못한 채로 기계적으로 사용하는 경우와 적용된 패턴을 이해하고 사용하는 경우는 큰 차이가 있다.
- 중복된 코드는 먼저 메서드로 분리하는 습관을 기르자.
- 그 중 일부 작업을 필요에 따라 바꾸어 사용해야 한다면 인터페이스를 사이에 두고 분리해서 전략 패턴을 적용하고 DI로 의존관계를 관리하도록 만든다.
- 그런데 바뀌는 부분이 한 애플리케이션 안에서 동시에 여러 종류가 만들어질 수 있다면 이번엔 **템플릿 / 콜백** 패턴을 적용하는 것을 고려해볼 수 있다.

#### 테스트와 try/catch/finally
- 가장 전형적인 템플릿 / 콜백 패턴의 후보
- 간단한 템플릿 / 콜백 예제 : 파일을 열어서 모든 라인의 숫자를 더 한 합을 돌려주는 코드
```java
// 테스트
...
public class CalcSumTest {
	@Test
	public void sumOfNumbers() throws IOException {
		Calculator calculator = new Calculator();
		int sum = calculator.calcSum(getClass().getResource(
			"numbers.txt").getPath());
		assertThat(sum, is(10));
	}
}
```

```java
public class Calculator {
	public Integer calSUm(String filepath) throws IOException {
		BufferedReader br = new BufferedReader(new FileReader(filePath));
		Integer sum = 0;
		String line = null;
		while(line = br.readLine() != null) {
			sum += Integer.valueOf(line);
		}
	
		br.close(); // 한 번 연 파일은 반드시 닫아준다.
		return sum;
	}
}
```
- calcSum() 메서드도 파일을 읽거나 처리하다가 예외가 발생하면 메서드를 빠져나가는 문제 발생
- try / finally 블록을 적용해서 어떤 경우에도 파일이 열렸으면 반드시 닫아줘야 한다.
- catch 블록으로 로그를 남기는 기능도 추가해보자.
```java
public Integer calSUm(String filepath) throws IOException {
	BufferedReader br = null;

	try {
		br = new BufferedReader(new FileReader(filePath));
		Integer sum = 0;
		String line = null;
		while(line = br.readLine() != null) {
			sum += Integer.valueOf(line);
		}
		return sum;
	}
	catch(IOException e) {
		System.out.println(e.getMessage());
		throw e;
	}
	finally {
		if (br != null) {
			try { br.close(); }
			catch(IOException e) { System.out.println(e.getMessage()); } 			}
	}
}
```
#### 중복의 제거와 템플릿 / 콜백 설계

- 반복되는 작업 흐름은 템플릿으로 독립
- 템플릿과 콜백의 경계를 정하고 템플릿이 콜백에게, 콜백이 템플릿에게 각각 전달하는 내용이 무엇인지 파악하는 것이 중요하다.
- 구조
	- `fileReadTemplate(String filePath, BufferedReaderCallback)` : 콜백 객체를 받아서 적절한 시점에 실행
   	- `BufferedReaderCallback` : 각 라인을 읽어서 처리한 후에 최종결과를 템플릿에 전달
```java
public interface BufferedReaderCallback {
  Integer doSomethingWithReader(BufferedReader br) throws IOException;
}          
public class Calculator {
  // BufferedReaderCallback을 사용하는 템플릿 메서드
  public Integer fileReadTemplate(String filePath, BufferedReaderCallback callback) throws IOException {
    BufferedReader br = null;
    try {
      br = new BufferedReader(new FileReader(filePath));
      return callback.doSomethingWithReader(br);
    } catch (IOException e) {
      throw e;
    } finally {
      if (br != null) { try { br.close(); } catch (Exception e) { throw e; } }
    }
  }
  // 템플릿/콜백을 적용한 calcSum 메서드
  public Integer calcSum(String filePath) throws IOException {
    BufferedReaderCallback callback = new BufferedReaderCallback() {
      @Override
      public Integer doSomethingWithReader(BufferedReader br) throws IOException {
        Integer sum = 0;
        String line = null;
        while((line = br.readLine()) != null) {
          sum += Integer.valueOf(line);
        }
        return sum;
      }
    };
    return fileReadTemplate(filePath, callback);
  }            
}
```

#### 템플릿 / 콜백 재설계
- 남아있는 중복되는 부분을 제거한다.
	```java
	public interface LineCallback {
	  Integer doSomethingWithLine(String line, Integer value);
	}
	public class Calculator {
	  // LineCallback을 사용하는 템플릿 메서드
	  public Integer lineReadTemplate(String filePath, LineCallback lineCallback, int initValue) throws IOException {
	    BufferedReader br = null;
	    try {
	      br = new BufferedReader(new FileReader(filePath));
	      Integer result = initValue;
	      String line = null;
	      while ((line = br.readLine()) != null) {
	        result = lineCallback.doSomethingWithLine(line, result);
	      }
	      return result;
	    } catch (IOException e) {
	      throw e;
	    } finally {
	      if (br != null) { try { br.close(); } catch (Exception e) { throw e; } }
	    }
	  }
	  // lineReadTemplate을 사용하도록 수정한 합, 곱 계산 메서드
	  public Integer calcMultiply(String filePath) throws IOException {
	    LineCallback sumCallback = new LineCallback() {
	      public Integer doSomethingWithLine(String line, Integer value) {
	        return value * Integer.valueOf(line);
	      }
	    };
	    return lineReadTemplate(filePath, sumCallback, 1);
	  }
	  public Integer calcSum(String filePath) throws IOException {
	    LineCallback sumCallback = new LineCallback() {
	      public Integer doSomethingWithLine(String line, Integer value) {
	        return value + Integer.valueOf(line);
	      }
	    };
	    return lineReadTemplate(filePath, sumCallback, 0);
	  }
	}
	```
- 제네릭스를 이용한 콜백 인터페이스
	```java
	public interface LineCallback<T> {
	  T doSomethingWithLine(String line, T value);
	}
	public class Calculator {
	  // 제너릭을 적용한 템플릿 메서드
	  public <T> T lineReadTemplate(String filePath, LineCallback<T> lineCallback, T initValue) throws IOException {
	    BufferedReader br = null;
	    try {
	      br = new BufferedReader(new FileReader(filePath));
	      T result = initValue;
	      String line = null;
	      while ((line = br.readLine()) != null) {
	        result = lineCallback.doSomethingWithLine(line, result);
	      }
	      return result;
	    } catch (IOException e) {
	      throw e;
	    } finally {
	      if (br != null) { try { br.close(); } catch (Exception e) { throw e; } }
	    }
	  }
	  public String concatenate(String filePath) throws IOException {
	    LineCallback<String> concatenateCallback = new LineCallback<String>() {
	      public String doSomethingWithLine(String line, String value) {
	        return value + line;
	      }
	    };
	    return lineReadTemplate(filePath, concatenateCallback, "");
	  }
	  public Integer calcMultiply(String filePath) throws IOException {
	    LineCallback<Integer> sumCallback = new LineCallback<Integer>() {
	      public Integer doSomethingWithLine(String line, Integer value) {
	        return value * Integer.valueOf(line);
	      }
	    };
	    return lineReadTemplate(filePath, sumCallback, 1);
	  }
	  public Integer calcSum(String filePath) throws IOException {
	    LineCallback<Integer> sumCallback = new LineCallback<Integer>() {
	      public Integer doSomethingWithLine(String line, Integer value) {
	        return value + Integer.valueOf(line);
	      }
	    };
	    return lineReadTemplate(filePath, sumCallback, 0);
	  }            
	}
	```
## 3.6 스프링의 JdbcTemplate
- 스프링은 다양한 템플릿 / 콜백 기술을 제공한다. 그 중 JdbcTemplate를 사용해본다.
```java
public class UserDao {
  private JdbcTemplate jdbcTemplate;
  public void setDataSource(DataSource dataSource) {
    this.jdbcTemplate = new JdbcTemplate(dataSource);
    this.dataSource = dataSource;
  }
}
```
- update()
  - deleteAll()에 적용
	```java
	public void deleteAll() throws SQLException {
	  this.jdbcTemplate.update("delete from users");
	}
  	```
  - add()에 적용
	```java
	public void add(User user) throws ClassNotFoundException, SQLException {
	  this.jdbcTemplate.update("insert into users(id, name, password) values(?, ?, ?)"
	      , user.getId(), user.getName(), user.getPassword());
	}
 	```
- queryForInt()
  - 콜백이 2개인 .query() 메서드 활용
	- 첫 번째 콜백 : statement 생성
	- 두 번째 콜백 : ResultSet으로 값 추출(제너릭)
		```java
		public int getCount() {
			  return this.jdbcTemplate.query(new PreparedStatementCreator() {
			    @Override
			    public PreparedStatement createPreparedStatement(Connection connection) throws SQLException {
			      return connection.prepareStatement("select count(*) from users");
			    }
			  }, new ResultSetExtractor<Integer>() {
			    @Override
			    public Integer extractData(ResultSet resultSet) throws SQLException, DataAccessException {
			      resultSet.next();
			      return resultSet.getInt(1);
			    }
			  });
			}
	  	```
- queryForInt() 활용
	```java
	// queryForInt는 spring 3.2.2에서 deprecated. 현재는 queryForObject 뿐
	public int getCount() {
	  return jdbcTemplate.queryForObject("select count(*) from users", Integer.class);
	}
	```
- queryForObject()
	- RowMapper : ResultSet 로우 하나를 매핑하기 위해 사용
	- 한 개의 결과만 얻을 것으로 기대
		- single일까 unique일까 -> unique!
		- 2개 이상이면 IncorrectResultSizeDataAccessException 발생 하면서 실패
		- 0개면 EmptyResultDataAccessException 발생 하면서 실패
   		```java
		public User get(String id) throws SQLException {
		  return jdbcTemplate.queryForObject("select * from users where id = ?",
		    // SQL에 바인딩 할 파라미터 값. 가변인자 대신 배열 사용
		    new Object[] { id },
		    // ResultSet 한 로우의 결과를 오브젝트에 매핑해주는 RowMapper콜백
		    new RowMapper<User>() {
		      public User mapRow(ResultSet rs, int rowNum) throws SQLException {
			User user = new User();
			user.setId(rs.getString("id"));
			user.setName(rs.getString("name"));
			user.setPassword(rs.getString("password"));
			return user;
		      }
		    });
		}
	  	```
- query()
	- public <T> List<T> query(String sql, RowMapper rowMapper)
	- 기능 정의와 테스트 작성
		- 모든 사용자 정보를 가져오기 위한 getAll은 id순으로 정렬해서 가져오도록 함
	- 테스트 코드
		```java
		@Test
		public void getAll() throws Exception {
		  userDao.deleteAll();
		  User user1 = new User("001", "name1", "password1");
		  User user2 = new User("002", "name2", "password2");
		  User user3 = new User("003", "name3", "password3");
		  List<User> originUserList = Arrays.asList(user1, user2, user3);
		  for(User user : originUserList) { userDao.add(user); }
		       List<User> userList = userDao.getAll();
		       assertThat(userList.size()).isEqualTo(originUserList.size());
		  for(int i = 0; i < userList.size(); i++) {
		    checkSameUser(originUserList.get(i), userList.get(i));
		  }
		}
		private void checkSameUser(User originUser, User resultUser) {
		  assertThat(originUser.getId()).isEqualTo(resultUser.getId());
		  assertThat(originUser.getName()).isEqualTo(resultUser.getName());
		  assertThat(originUser.getPassword()).isEqualTo(resultUser.getPassword());
		}
		```
	- UserDao 코드
		```java
		public List<User> getAll() {
		  return jdbcTemplate.query("select * from users order by id",
		    (rs, rowNum) -> {
		      User user = new User();
		      user.setId(rs.getString("id"));
		      user.setName(rs.getString("name"));
		      user.setPassword(rs.getString("password"));
		      return user;
		    });
		}
		```
	- 테스트 보완
		- 네거티브 테스트 필요
		- 아래처럼 빈 테이블의 결과는 size가 0인 List이다 라고 알 수 있는 테스트가 있으면, getAll메서드가 내부적으로 어떻게 작동하는지 알 필요도 없어지는 장점이 있다.
		```java
		@Test
		public void getAllFromEmptyTable() throws Exception {
		  userDao.deleteAll();
		  List<User> userList = userDao.getAll();
		  assertThat(userList.size()).isEqualTo(0);
		}
		```

  - 재사용 가능한 콜백의 분리
  - DI를 위한 코드 정리
    ```java
    // 불필요한 DataSource, JdbcContext 제거
	public class UserDao {        
	  private JdbcTemplate jdbcTemplate;        
	  public UserDao(DataSource dataSource) {
	    this.jdbcTemplate = new JdbcTemplate(dataSource);
	}
 	```
- 중복 제거
  - get(), getAll()에 RowMapper의 내용이 똑같음
  - 단 두개의 중복이라 해도 언제 어떻게 확장이 필요해질지 모르니 제거하는게 좋다
  	```java
	  // 중복 제거한 rowMapper와 get()메서드
	public User get(String id) {
	  return jdbcTemplate.queryForObject("select * from users where id = ?",
		  new Object[] { id },
		  getUserMapper());
	}          
	private RowMapper<User> getUserMapper() {
	  return (resultSet, i) -> {
	    User user = new User();
	    user.setId(resultSet.getString("id"));
	    user.setName(resultSet.getString("name"));
	    user.setPassword(resultSet.getString("password"));
	    return user;
	  };
	}
	```

- 템플릿/콜백 패턴과 UserDao
  - 템플릿/콜백 패턴과 DI를 이용해 깔끔해진 UserDao 클래
  - 응집도 높다 : 테이블과 필드정보가 바뀌면 UserDao의 거의 모든 코드가 함께 바뀐다
  - 결합도 낮다 : JDBC API 활용방식, 예외처리, 리소스 반납, DB 연결 등에 대한 책임과 관심은 JdbcTemplate에 있기 때문에 변경이 일어난다 해도 UserDao의 코드에는 영향을 주지 않는다
	```java
	public class UserDao {          
	private JdbcTemplate jdbcTemplate;          
	public UserDao(DataSource dataSource) {
	this.jdbcTemplate = new JdbcTemplate(dataSource);
	}          
	public void add(final User user) {
	this.jdbcTemplate.update("insert into users(id, name, password) values(?, ?, ?)"
	, user.getId(), user.getName(), user.getPassword());
	}          
	public User get(String id) {
	return jdbcTemplate.queryForObject("select * from users where id = ?"
	    ,new Object[] { id }, getUserMapper());
	}
	private RowMapper<User> getUserMapper() {
	return (resultSet, i) -> {
	User user = new User();
	user.setId(resultSet.getString("id"));
	user.setName(resultSet.getString("name"));
	user.setPassword(resultSet.getString("password"));
	return user;
	};
	}
	public User getUserByName(String name) {
	return jdbcTemplate.queryForObject("select * from users where name = ?"
	     , new Object[] { name }, getUserMapper());
	}          
	public void deleteAll() {
	this.jdbcTemplate.update("delete from users");
	}          
	public int getCount() {
	return jdbcTemplate.queryForObject("select count(*) from users", Integer.class);
	}          
	public List<User> getAll() {
	return jdbcTemplate.query("select * from users order by id", getUserMapper());
	}          
	}
	```

## 3.7 정리
예외처리와 안전한 리소스 반환을 보장해주는 DAO 코드를 만들었다.
객체지향 설계 원리, 디자인 패턴, DI 등을 적ㅇ요해서 깔끔하고 유연하며 단순한 코드로 만들었다.
JDBC 처럼 예외발생 가능성이 있으며 공유 리소스 반환이 필요한 코드는 반드시 try-catch-finally 블록으로 관리해야 한다.
전략 패턴
로직에 반복이 있으면서 그 중 일부(전략)만 바뀌고 일부(컨텍스트)는 바뀌지 않는다면 전략 패턴을 적용한다.
```
한 애플리케이션 안에서 여러 전략을 동적으로 구성하고 사용해야 한다면 컨텍스트를 이용하는 클라이언트 메서드에서 직접 전략을 정의하고 제공하도록 만든다.
익명 내부 클래스를 사용해서 전략 오브젝트를 구현하면 편리하다.
컨텍스트가 하나 이상의 클라이언트 객체에서 사용된다면 클래스를 분리해서 공유하도록 만든다.
컨텍스트는 별도 빈으로 등록해서 DI 받거나 클라이언트 클래스에서 직접 생성해 사용한다.
```
템플릿 콜백 패턴
단일 전략 메서드를 갖는 전략 패턴이면서, 익명 내부 클래스를 사용해서 매번 전략을 새로 만들어 사용하고, 컨텍스트 호출과 동시에 전략 DI를 수행하는 방식
콜백 : 다시 불려지는 기능 이라는 의미
```
콜백 코드에도 일정한 패턴이 반복된다면 콜백을 템플릿에 넣고 재활용 하는 것이 편리하다.
템플릿과 콜백의 타입이 다양하게 바뀔 수 있다면 제너릭을 이용한다.
스프링은 JDBC 코드 작성을 위해 JdbcTemplate을 기반으로 하는 다양한 템플릿과 콜백을 제공한다.
템플릿은 한 번에 하나 이상의 콜백을 사용할 수도 있고, 하나의 콜백을 여러번 호출할 수 있다.
템플릿/콜백을 설계할 때에는 템플릿과 콜백 사이에 주고받는 정보에 관심을 두어야 한다.
```
템플릿/콜백은 스프링이 객체지향 설계와 프로그래밍에 얼마나 가치를 두고 있는지를 잘 보여주는 예다.
얼마나 유연하고 변경이 용이하게 만들고자 하는지를 잘 보여주는 예다.
이 챕터에서는 추상화(템플릿), 캡슐화(DI)를 통해서
DI에 녹아있는 캡슐화? -> DataSource가 갖는 세세한 정보를 DataSource를 직접 사용하는 UserDao는 알 필요가 없기 때문
중첩 클래스의 종류
스태틱 클래스(static class)
독립적으로 오브젝트로 만들어질 수 있는 클래스
내부 클래스(inner class)
자신이 정의된 클래스와 오브젝트 안에서만 만들어 질 수 있는 클래스
멤버 내부 클래스 : 멤버 필드처럼 오브젝트 레벨에 정의됨
로컬 클래스 : 메서드 레벨에 정의됨
익명 내부 클래스 : 이름을 갖지 않으며 선언된 위치에 따라 범위가 다름
