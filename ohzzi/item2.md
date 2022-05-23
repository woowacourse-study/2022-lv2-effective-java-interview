# [아이템 2] 생성자에 매개변수가 많다면 빌더를 고려하라

## 주제 선정의 이유

미션을 진행하다 보니 도메인이 너무 많은 필드를 가지고 있어 생성자의 유지보수에 어려움을 겪었습니다.

```java
public class Section {
    private final Long id;
    private final Station upStation;
    private final Station downStation;
    private final int distance;
}
```

위와 같은 도메인의 생성자를 호출할 때, `upStation`과 `downStation`이 뒤바뀌거나 한다면?

```java
Section 강남_역삼 = new Section(1L, 강남, 역삼, 1);
Section 강남_역삼 = new Section(1L, 역삼, 강남, 1);
```

둘 사이의 구분을 하기가 힘듭니다. 어떤 매개변수 자리에 어떤 값이 들어가야 하는지 헷갈릴 가능성이 크죠.

지금은 비록 생성자 매개변수가 4개라 쉽게 이해할 수 있고, IDE의 도움을 받으면 좀 더 수월하게 매개변수를 집어넣을 수도 있지만 필드가 늘어난다면 가독성은 점점 더 안좋아질 것입니다.

게다가 이번 미션을 진행하면서 DB 저장 전에는 id 값이 비어 있는 객체를, DB 저장 후에는 id 값이 들어 있는 객체를 생성해야 하니 자동적으로 `점층적 생성자 패턴`까지 사용하게 되면서, 생성자가 많아지고 관리하기 어렵게 되었습니다.

어떻게 하면 객체 생성의 가독성과 유지보수를 챙길 수 있을까 하다가 이펙티브 자바에서 읽은 빌더 패턴에 대해 생각이 났습니다.

## 빌더 패턴

빌더 패턴은 GoF 디자인 패턴 중의 하나입니다. 빌더 패턴은 `점층적 생성자 패턴`의 안정성과 `자바 빈즈 패턴`의 가독성을 겸비한 패턴으로, 클라이언트가 필요한 객체를 직접 만드는 대신, 필수 매개변수 만으로 생성자를 호출해 빌더 객체를 얻고, 빌더 객체가 제공하는 일종의 세터 메서드들로 원하는 선택 매개변수들을 설정한 뒤, 마지막으로 build 메서드를 호출해 객체(일반적으로는 불변인)를 얻는 방식입니다.

예를 들어 앞선 Section 클래스를 빌더 패턴의 형태로 바꿔본다면,

```java
public class Section {
    private final Long id;
    private final Station upStation;
    private final Station downStation;
    private final int distance;
    
    private Section(Builder builder) {
        this.id = builder.id;
        this.upStation = builder.upStation;
        this.downStation = builder.downStation;
        this.distance = builder.distance;
    }
    
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private Long id;
        private Station upStation;
        private Station downStation;
        private int distance;
        
        private Builder() {
        }
        
        public Builder id(Long id) {
            this.id = id;
            return this;
        }
        
        public Builder upStation(Station upStation) {
            this.upStation = upStation;
            return this;
        }
        
        public Builder downStation(Station downStation) {
            this.downStation = downStation;
            return this;
        }
        
        public Builder distance(int distance) {
            this.distance = distance;
            return this;
        }
        
        public Section build() {
            return new Section(id, upStation, downStation, distance);
        }
    }
}
```

(책에서는 Builder의 생성자를 public으로 만들고 원본 객체의 new 키워드와 Builder의 생성자를 함께 사용하는 방식으로 사용했지만, 저는 생성하고자 하는 객체 내에 `builder()` 라는 빌더를 반환해주는 정적 팩터리 메서드를 만들어주고 해당 메서드를 호출하는 것이 new 키워드의 사용도 없앨 수 있고 가독성도 더 좋다고 생각하기 때문에 이와 같은 방법을 사용합니다.)

와 같이 작성하고,

```java
Section 강남_역삼 = Section.builder()
                .id(1L)
                .upStation(강남)
                .downStation(역삼)
                .distance(1)
                .build();
```

와 같이 객체를 생성해 줄 수 있습니다. 만약 DB에 저장하기 전이라서 id 값이 없는 객체라면

```java
Section 강남_역삼 = Section.builder()
                .upStation(강남)
                .downStation(역삼)
                .distance(1)
                .build();
```

이와 같이 id 라는 메서드를 호출하지 않으면 됩니다.

## 빌더 패턴을 적용해 보니 어떤 장점이 있었나요?

확실히 빌더 패턴은 유연합니다. 일단 빌더를 만들어 놓기만 하면, 필드가 늘어나도 빌더 내에 메서드를 하나 더 추가해주고 원본 객체의 생성자를 조금 수정해 주기만 하면 됩니다. 또한 무엇보다 각각의 매개변수를 집어넣는 과정에서 메서드 명으로 매개변수를 구분할 수 있기 때문에, 가독성 측면에서 생성자에 이름을 붙일 수 있다는 정적 팩터리 메서드보다도 더 큰 장점을 가지고 있습니다.

또한 위에 보이는 `Section` 예제처럼 필요하지 않은 매개변수는 기본 값으로 놔두고 생성하기에도 유용합니다. 점층적 생성자 패턴 역시 필요하지 않은 매개변수를 배제하고 생성할 수 있지만, 반드시 초기화 할 필요가 없는 필드가 많아진다면 생성자의 개수가 늘어남에 따라 가독성 및 유지보수 측면에서 손해를 많이 볼 것입니다.

## 그럼 빌더 패턴은 단점이 없나요?

아니요. 일단 굉장히 귀찮습니다. 어쩌면 그냥 생성자를 계속 많이 만들어주는 것 보다 저렇게 빌더 클래스를 만들어 주는 것이 더 복잡하게 보일 정도로요. 일단 만들어 놓고 나면 가독성과 유지보수 측면에서는 장점이 어마어마하지만, 만들기까지의 과정이 매우 험난합니다. 물론 `Lombok`이라는 굉장히 편리한 라이브러리를 통해 `@Builder` 어노테이션 하나를 붙이는 것 만으로도 해결할 수 있기는 합니다.

그리고 위 코드에서 단점을 찾자면, 제가 깜빡하고 빌더에서 `upStation()` 메서드나 `downStation()` 메서드의 호출을 까먹는다면, null이 들어가면 안되는 자리에 null이 들어가버린다는 단점도 있습니다. 해당 필드들은 필수 필드이기 때문이죠. 원래 이펙티브 자바 책에서는 필수 필드들은 빌더의 생성자에서 바로 초기화하라고 되어 있는데, 이렇게 되면 `upStation`과 `downStation` 위치의 가독성을 확보하려는 빌더 사용의 의미가 퇴색됩니다.

## 그래서 결론적으로 빌더를 쓰셨나요, 안쓰셨나요?

원래는 적용했었습니다. 그러나 설계상의 이유로 git을 reset 해버리고 다시 코드를 짜는 과정에서, 다시 빌더 클래스를 만들 엄두가 나지 않았습니다. 게다가 빌더 패턴으로 바꾼다면 기존 테스트 코드 등에 들어있는 Section의 생성자를 모두 빌더로 바꿔줘야 한다는 점도 저를 망설이게 했습니다.

Station이 막 3개, 4개씩 들어가는 클래스였다면 그래도 빌더를 사용했겠지만, 저 Section 클래스가 빌더 클래스를 만드는 노력을 감수할만한 클래스라고 생각하지는 않았습니다. 당시에는 기간에 치여서 우선 미션을 완성하는 것이 가장 중요한 과제였기 때문입니다. 때문에 이후의 코드에서는 빌더를 사용하지 않았습니다.

하지만 매개변수가 더 늘어난다거나 하면 빌더 클래스를 충분히 고려해볼만 한 것 같습니다. 그리고 만약 제게 미션을 마무리할 시간이 더 많았다면 충분히 빌더 패턴을 다시 적용했을 것 같습니다. 가독성과 유지보수 측면에서 장점을 무시할 수가 없거든요.

가장 좋은 방법은 애초에 설계상 필드가 많아질 것 같다면 시작부터 빌더 패턴으로 설계를 해두는 것이라고 생각합니다. 생성자를 빌더 패턴으로 죄다 바꿔줄 일도 없어지고, 필드를 추가하기도 더 쉬워지기 때문입니다. 아니면, Lombok의 사용을 고려해 보는 것도 좋은 방법이라고 생각합니다.
