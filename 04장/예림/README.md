# 4장. 예외
- 예외와 관련된 코드는 자주 엉망이 되거나 무성의하게 만들어지기 쉽다.
- 때론 잘못된 예외처리 코드 때문에 찾기 힘든 버그를 낳을 수도 있고, 생각지 않았던 예외상황이 발생했을 때 상상 이상으로 난처해질 수도 있다.

## 1.사라진 SQLException
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
- 어째서 내부적으로 JDBC API를 쓰는 jdbcTemplate가 SQLException을 바깥으로 던지지 않는가?
### 1-1. 초난감 예외 처리


