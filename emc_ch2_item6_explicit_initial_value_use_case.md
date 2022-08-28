# 항목 6: auto가 원치 않은 형식으로 연역될 때에는 명시적 형식의 초기치를 사용하라
auto를 사용하면 여러가지 장점이 있으나, 가끔은 auto의 형식 연역이 딴전을 부리기도 한다.  
vector<bool> 컨테이너를 auto 변수로 초기화할 경우를 예로 살펴보자.
~~~C++
// Widget을 하나 받고 std::vector<bool>을 돌려주는 함수
// Widget이 특정 기능(feature)을 지원하는지를 return함.
std::vector<bool> features(const Widget& w);

Widget w;
//...
bool highPriority = features(w)[5]; // 5번째 원소는 우선순위를 의미한다고 가정
//...
processWidget(w, highPriority);   // w를 그 우선순위에 맞게 처리한다.
~~~

위의 코드는 정상적으로 동작하지만, highPriority를 auto로 선언할 경우 미정의 동작을 유발한다.  
  
~~~C++
// Widget을 하나 받고 std::vector<bool>을 돌려주는 함수
// Widget이 특정 기능(feature)을 지원하는지를 return함.
std::vector<bool> features(const Widget& w);

Widget w;
//...
auto highPriority = features(w)[5]; // 5번째 원소는 우선순위를 의미한다고 가정
//...
processWidget(w, highPriority);   // w를 그 우선순위에 맞게 처리한다.
~~~
 - auto를 사용할 경우, highPriority의 형식은 bool이 아니기 때문이다.  
std::vector<bool>의 operator[]가 돌려주는 것은 그 컨테이너의 한 요소에 대한 참조가 아니라, std::vector<bool>::reference 형식(std::vector<bool> 안에 내포된 클래스)의 객체이다.  

 - auto로 선언했을 경우와 bool로 명시적으로 선언한 경우의 동작과정을 살펴보자.  
    - highPriority변수를 bool을 사용해서 선언했을 경우
      - features는 std::vector<bool> 객체를 돌려주며, 그 객체에 대해 operator[]가 호출된다.
      - operator[]는 std::vector<bool>::reference 객체를 돌려주며, 그 객체가 암묵적으로 bool로 변환되어서 highPriority의 초기화에 쓰인다.
      - 결과적으로 std::vector<bool>의 5번 원소의 값을 갖게 된다.(우리가 원했던 결과다.)

    - highPriority변수를 auto를 사용해서 선언했을 경우
      - features는 std::vector<bool> 객체를 돌려주고, 그 객체에 대해 operator[]가 호출된다.  
      - 그 operator[]가 std::vector<bool>::reference 객체를 돌려준다는 점도 같다. 그러나 auto에 의해 highPriority의 형식이 연역되기 때문에, highPriority는 bool로 암시적으로 변환되지 않고 std::vector<bool>::reference 객체의 한 복사본의 5번째 원소가 된다.  
      - 그런데, features()의 반환값(std::vector<bool>)은 임시 객체이기 때문에 operator[]의 반환값인 std::vector<bool>::reference 객체는 임시 객체에 대한 참조를 담고 있다.  
      임시객체이므로 문장의 끝에서 파괴되게 되고,, highPriority의 포인터는 대상을 잃은 포인터가 된다(dangling pointer).  
      (값이 아닌 참조를 반환하기 때문에 dangling pointer를 참조하게 된다.)  
      결국 processWidget 호출은 미정의 행동을 유발하게 된다.  

## 대리자(proxy class)
 - std::vector<>::operator[]()가 반환하는 것은 std::vector<>::reference라는 대리자 클래스 이다.  
 - 대리자(proxy) 설계 패턴은 소프트웨어 전반적으로 널리 사용되는 패턴 중의 하나로, 표준 템플릿 라이브러리에서도 자주 활용되는 디자인패턴이다.  
 - std::shared_ptr, std::unique_ptr도 대리자클래스로 구현되어 있다.
 - 대리자(proxy) 설계 패턴은 실제 구현 목적을 흉내낸 '대리자' 역할으 하는 것인데,
 unique_ptr, shared_ptr의 경우는 클라이언트에게 명벡히 드러나도록 설계되었으나,  
 std::vector<bool>::reference의 경우, '보이지 않는' 대리자 이므로 다소 은밀하게 동작되도록 설계되었다. 이런 경우, 주의해서 auto를 사용하는 것이 좋다.
  - 이런 경우, 아래와 같이 static_cast를 auto와 함께 사용하는 것이 좋다.
~~~C++
auto highPriority = static_cast<bool>(features(w)[5]);
~~~
