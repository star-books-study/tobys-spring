# 3장. 템플릿

## 3.1 다시보는 초난감 DAO

- DB 커넥션처럼 제한적인 리소스를 공유해 사용하는 서버에서 동작하는 JDBC 코드는 예외처리가 반드시 필요하다. 예외가 발생하더라도 사용한 리소스를 반드시 반환하도록 만들어야 한다. 그렇지 않다면 어느 순간에는 커넥션 풀의 여유가 없어지고 리소스가 모자라며 서버가 터질 수 있다.



        public void deleteAll() throws SQLException {
        Connection c = dataSource.getConnection();
                PreparedStatement ps = c.prepareStatement("delete from users");
                ps.executeUpdate();

                ps.close();
                c.close();
    }
- 언뜻 문제가 없어보이지만, preparedStatement 처리하는 중 예외가 발생하면 자원을 반환하는 close() 메서드가 수행되지 않아 반환되지 않을 수 있다.

- 그래서 예외가 발생하더라도 리소스를 반환하도록 수정하여야 한다.