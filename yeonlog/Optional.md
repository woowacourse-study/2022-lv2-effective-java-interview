## 주제 선택 사유

DAO를 생성할 때 반환 타입으로는 어떤 것이 좋을까? 라는 생각에서 선정하게 되었습니다

<br>

## null이 아닌 빈 컬렉션 또는 배열을 반환하라.

> 길이가 0인 경우 null을 반환한다면?

- 객체가 0일 가능성이 거의 없는 상황에서는 오류 상황이 수년 뒤에야 발견될 수 있음
- null을 반환하는 메서드 사용 시 방어 코드를 항상 고려해야 함
    - 코드의 중복
    - 로직의 복잡도 ↑
    - 가독성 ↓

<br>

### 빈 컨테이너를 할당하는건 비용이 든다?

👉 빈 컬렉션/배열을 할당하는데 비용이 든다는건 맞는 말이다.

하지만 이 정도 성능 차이는 신경을 써야할 수준은 아니다.  
**빠른 프로그램보다 좋은 프로그램을 만들자**

게다가 빈 컬렉션/배열은 새로 할당하지 않아도 반환할 수 있다.

```java
public class Cars {

    private final List<Car> value;

    // AS-IS
    public List<Car> getValueByNull() {
        if (value.isEmpty()) {
            return null;
        }
        return new ArrayList<>(value);
    }

    // TO-BE
    public List<Car> getValue() {
        return new ArrayList<>(value);
    }
}
```

성능 차이가 궁금해서 `getValueByNull`와 `getValue`를 100000번 호출해서 시간을 측정해보았다. 
측정할 때마다 다른 결과를 보인다.

```
# try 1
getValueByNull: 16
getValue: 15

# try 2
getValueByNull: 15
getValue: 16

# try 3
getValueByNull: 10
getValue: 16

# try 4
getValueByNull: 0
getValue: 0
```

isEmpty() 메서드로 인한 차이인가 싶어서 무조건 null을 반환하도록 시도해았지만 마찬가지로 결과가 항상 랜덤했다.

<br>

### 빈 불변 컬렉션 반환하고 싶다면?

`Collections.emptyList`, `Collections.emptyMap` 등을 활용할 수도 있다. 
이 메서드를 꼭! 사용 해야한다는 의미는 아니다. 
빈 불변 컬렉션 반환에 대해 최적화가 필요한 경우 도입하고 실제로 성능이 개선 되는지도 확인해보아야 한다.

<br>

### jdbcTemplate은 어떤 값을 반환하고 있을까?

> 현재 제가 자주 사용하고 있는 `NamedParameterJdbcTemplate`를 이용해 디버깅 해보았습니다.

```java
public class NamedParameterJdbcTemplate implements NamedParameterJdbcOperations {

    // ...

    @Override
    public <T> List<T> query(String sql, SqlParameterSource paramSource, RowMapper<T> rowMapper)
            throws DataAccessException {

        return getJdbcOperations().query(getPreparedStatementCreator(sql, paramSource), rowMapper);
    }
}
```

```java
public class RowMapperResultSetExtractor<T> implements ResultSetExtractor<List<T>> {

    // ...

    @Override
    public List<T> extractData(ResultSet rs) throws SQLException {
        List<T> results = (this.rowsExpected > 0 ? new ArrayList<>(this.rowsExpected) : new ArrayList<>());
        int rowNum = 0;
        while (rs.next()) {
            results.add(this.rowMapper.mapRow(rs, rowNum++));
        }
        return results;
    }
}
```

<br><br>

## null이 아닌 Optional을 반환하라.

Optional은 선택형 값을 캡슐화하는 클래스다. 
값이 없을 수 있음을 명시적으로 보여준다. 
API 사용자는 Optional가 return되는 것을 보면 null일 가능성도 존재하구나!를 인식할 수 있게 된다.  
👉 이해하기 쉬운 API를 설계할 수 있도록 돕는다.

<br>

### Optional 객체 만들기

- `Optional.empty()`: 빈 Optional 객체 생성
- `Optional.of(value)`: null이 아닌 값을 포함하는 Optional 객체 생성
- `Optional.ofNullable(value)`: null일 수도 있는 값을 포함하는 Optional 객체 생성

<br>

### 반환이 아닐 때도 Optional을 쓰면 좋을까?

결론부터 말하자면 **NO** 다.

Java 설계자인 Brian Goetz이 정의한 Optional을 살펴보자.

> Optional is intended to provide a limited mechanism for library method return types where there needed to be a clear way to represent “no result," and using null for such was overwhelmingly likely to cause errors.

요약해보자면 `Optional의 용도는 선택형 반환값`을 지원하는 것이다. 
null 반환 시 오류 발생 가능성이 높은 경우에 '결과 없음'을 명확하게 드러내고, 
메서드의 반환 타입으로만 사용되도록 매우 제한적이게 만들어졌다.

예제로 어떤 클래스의 필드에 null일 수도 있을 경우를 대비해 Optional을 사용한다고 가정해보자. 
Optional은 필드 형식으로 사용할 것을 가정하지 않았다. 
따라서 직렬화 시 필요한 `Serializable` 인터페이스를 구현하지 않았다. 
해당 클래스를 직렬화할 일이 생긴다면 오류가 날 것이다.

물론 도메인 모델에 Optional을 사용하는 것이 바람직한 경우는 존재는 한다. 
(ex: 객체 그래프에서 일부 또는 전체가 null일 수 있는 상황)

> 객체 그래프: 여러 객체 간의 관계를 가리키는 가리키는 그래프

Optional을 파라미터로 사용하는 경우에도 노란색 밑줄로 경고가 하나 뜬다. 
위에서 설명했던 것들과 동일하게 `설계 의도와는 다른 목적으로 사용하지 마!` 는 맥락에서 Optional을 권장하지 않는다. 
좀 더 구체적인 이유가 궁금하다면 [블로그 글](https://yeonyeon.tistory.com/224) 을 참고하길 바란다.

<br>

### JdbcTemplate은 Optional을 반환하지 않는다.

우리는 sql의 결과가 단 하나라고 생각되는 경우 JdbcTemplate의 queryForObject을 이용한다. 
해당 메서드로 가보면 결과 값이 없는 경우 EmptyResultDataAccessException, 결과 값이 여러개인 경우 IncorrectResultSizeDataAccessException를 반환한다. 
만약 Optional 형태로 반환하고 싶다면 `try-catch`로 감싸서 직접 결과값을 지정해줘야 한다.

```java
public class JdbcTemplate extends JdbcAccessor implements JdbcOperations {

    @Override
    @Nullable
    public <T> T queryForObject(String sql, Object[] args, int[] argTypes, RowMapper<T> rowMapper)
            throws DataAccessException {

        List<T> results = query(sql, args, argTypes, new RowMapperResultSetExtractor<>(rowMapper, 1));
        return DataAccessUtils.nullableSingleResult(results);
    }
}
```

```java
public abstract class DataAccessUtils {
    @Nullable
    public static <T> T nullableSingleResult(@Nullable Collection<T> results)
            throws IncorrectResultSizeDataAccessException {
        // This is identical to the requiredSingleResult implementation but differs in the
        // semantics of the incoming Collection (which we currently can't formally express)
        if (CollectionUtils.isEmpty(results)) {
            throw new EmptyResultDataAccessException(1);
        }
        if (results.size() > 1) {
            throw new IncorrectResultSizeDataAccessException(1, results.size());
        }
        return results.iterator().next();
    }
}
```

<br><br>

## 결론

null을 반환하는 것은 호출하는 쪽에서 조건부 로직을 유도한다. 
반드시 null 체크를 해줘야하고 이로 인해 생기는 if 결과에 따라 로직이 둘로 나뉘어야 한다.

DAO에서 sql 호출에 대한 결과값을 DB에서 가져온다. 
직접 만든 API와는 달리 어떤 결과 값을 가져올지 예상하지 못한다. 
항상 null 체크를 해주기 보다는 빈 컬렉션, Optional 등을 활용하도록 하자.

<br><br>

***

### 참고

- Effective Java / 조슈아 블로크
    - item 54: null 이 아닌, 빈 컬렉션이나 배열을 반환하라
    - item 55: 옵셔널 반환은 신중히 하라.
- 모던 자바 인 액션 / 라울 게이브리얼 우르마, 마리오 푸스코, 앨런 마이크로프트
    - chapter 11: null 대신 Optional 클래스
