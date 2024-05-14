# 4장. 예외
## 4.1. 사라진 SQL EXCEPTION
### 4.1.1. 초난감 예외처리
#### 예외 블랙홀
```java
try {
    ...
} catch(SQLException e) {

}
```
- 예외를 잡고 아무것도 하지 않는 코드이다.