# [아이템75] 예외의 상세 메시지에 실패 관련 정보를 담으라

## Summary

예외의 상세 메세지에 실패 원인에 대한 자세한 정보를 담아야 하는 이유

- 실패 원인 분석을 위한 정보를 예외 메세지가 아니면 알기 어렵기 때문
- 재현하기 어려운, 우연적인 예외 발생 상황에서는 더더욱!

<br/>

어떤 정보를 담아줘야 할까?

- 예외 발생에 관여된 모든 매개변수와 필드의 값
- 보안에 영향을 끼치는 정보를 담지 않도록 주의

<br/>

여기서 말하는 상세 메세지 ≠ 사용자에게 보여줄 오류 메세지

- 사용자에게는 가독성 좋고 친절한 메세지
- 상세 메세지의 주 소비층은 프로그래머와 엔지니어
    - 문제 분석을 위한 내용을 잘 담아주어어야

<br/>

필요한 정보를 예외 생성자의 parameter를 통해 받아서 메세지 생성하기

- 상세 메세지 작성하는 코드가 클래스 내부에 위치하게 됨
    - 생성자 호출 시 메세지 만드는 작업 반복하지 않아도 된다

<br/>

## 그래서, 미션에 어떻게 적용했나요?

> Exception 생성자 재정의는 안했습니다.
Exception의 생성자를 다시 정의하려면, Custom Exception을 사용해야 할 것인데....

#### Custom Exception을 사용하는 이유가, 자세한 예외 message만으로 충분한가?
- 표준 예외를 사용하지 않을 경우, 동료 개발자 입장에서는 해당 예외가 의미하는 context에 대해 따로 파악해야 한다
- 관리해야할 class가 늘어난다

#### 현재 프로그램 상에서 자세한 예외 message가 이러한 단점을 감수할 정도로 필요한가?
- 현재 내가 발생시키는 예외 상황은 그리 deep한 debuging을 요구하지 않는다
    - 경험 상 현재 정의된 예외 상황에 대해서 만큼은 exception message를 보는 것 이외에 다른 디버깅이 필요하지 않았다
    - 그렇다면 이미 나의 예외 message는 충분한 정보를 담고 있다
- 그렇다고 사용자에게 보여주기에 가독성이 떨어지는 message도 아니다
    - 그렇다면 굳이 message를 분리할 필요도 없다

#### 더 자세한 정보를 주면 좋긴 하지 않은가?
- 그것은 굳이 생성자를 재정의하지 않고도 아래와 같이 보여줄 수 있다.


```java
public void deleteById(Long id) {
    final String sql = "DELETE FROM station WHERE id = ?";
    final int deletedCount = jdbcTemplate.update(sql, id);
    validateRemoved(deletedCount, id);
}

private void validateRemoved(int deletedCount, Long id) {
    if (deletedCount == 0) {
        throw new IllegalStateException("삭제하고자 하는 역이 존재하지 않습니다. : " + id);
    }
}
```
