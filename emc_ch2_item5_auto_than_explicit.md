# 항목 5: 명시적 형식 선언보다는 auto를 선호하라.
C++11부터는 auto가 지원된다.  
auto 변수의 형식은 해당 초기치로부터 연역되므로, 반드시 초기화를 해야한다.  
auto를 사용하게 되면 변수의 초기화가 되지 않아서 쓰레기값으로 초기화 될 일이 없다는 장점이 있다.  
~~~C++
int x1;       // x1에 쓰레기값이 들어갈 수 있다.
auto x2;      // 컴파일 에러. 초기치가 꼭 필요하다.
auto x3 = 0;  // 양호함: x3의 값이 잘 정의됨.
~~~
  
auto의 몇가지 사용 예들..  
~~~C++
// 반복자 역참조
template<typename It>
void dwim(It b, It e)
{
  for (; b != e; ++b) {
    auto currValue = *b;
    ...
  }
}

// 람다 함수를 담는 변수 선언(C++11)
auto derefUPLess =                        // std:: unique_ptr들이 가리키는
  [](const std::unique_ptr<Widget>& p1,   // Widget 객체들을 비교하는 함수
      const std::unique_ptr<Widget>& p2)
      { return *p1 < *p2; };

// C++14 버전
auto derefLess =                          // 그 어떤 것이든 포인터처럼 동작하는 것들이
  [](const auto& p1,                      // 가리키는 값들을 비교하는 함수
      const auto& p2)
      { return *p1 < *p2; };
~~~
  
클로저를 담는 변수를 선언 시에 std::function을 사용할 수도 있지만,  
위에서처럼 람다(클로저)를 담는 변수를 선언할 때 auto를 사용하면 얻는 이점이 있다.  
먼저 std::function에 대해 알아보자.  

## std::function
 - std::function은 C++11 표준 라이브러리이며, 함수포인터 개념을 일반화한 것이다.  
 - 단, 함수포인터는 함수만 가리킬 수 있지만 std::function은 호출 가능한 객체까지 가리킬 수 있다.  
 - 함수포인터를 만들 때 그 포인터가 가리키는 함수의 형식(함수의 서명)을 지정해야하는 것과 마찬가지로,  
std::function객체를 생성할 때에는 그것이 지칭할 함수의 형식을 반드시 지정해야한다.  
함수의 형식은 std::function의 매개변수로 설정이 가능하다.
예를 통해 확인해보자.  
~~~C++
bool(const std::unique_ptr<Widget>&,  // C++11 버전
      const std::unique_ptr<Widget>&) // std::unique_ptr<Widget> 비교함수의 서명

// 위의 서명을 전달하도록 하는 std::function 객체 선언.
std::function<bool(const std::unique_ptr<Widget>&,
                    const std::unique_ptr<Widget>&)> func;

// 위의 절에서 본 derefUPLess함수를 auto를 사용하지 않고 선언한다면..
// 아래처럼 std:function으로 선언이 가능하다.
std::function<bool(const std::unique_ptr<Widget>&,
                    const std::unique_ptr<Widget>&)>
  derefUPLess =   [](const std::unique_ptr<Widget>& p1,
                      const std::unique_ptr<Widget>& p2)
                      { return *p1 < *p2; };
~~~

 - auto를 사용해서 클로저 변수를 선언 시의 장점  
auto를 사용하면 더 깔끔해보인다. 그런데 이 점말고도 더 중요한 포인트가 있다.  
auto로 선언된, 그 클로저를 담는 변수는 그 클로저와 같은 형식이며,  
따라서 그 클로저에 요구되는 만큼의 메모리만 사용한다.  

 - std::function을 사용해서 클로저 변수를 선언하면..  
    - std::function으로 선언된 변수의 형식은 std::function 템플릿의 한 인스턴스 이다.
    - 그 크기는 임의의 주어진 서명에 대해 고정되어 있다.
    - 그 크기가 요구된 클로저를 저장하기에 부족할 수도 있다.  
    - 그런 경우, std::function은 힙메모리를 할당해서 클로저를 저장한다.  
    - 결과적으로, std::function 객체는 auto로 선언된 객체보다 메모리를 더 많이 소비할 수 있다.  
    - 그리고 인라인화를 제한하고 간접 함수 호출을 산출하는 구현 세부사항 때문에,  
       std::function 객체를 통해서 클로저를 호출하는 것은 auto로 선언된 객체를 통해서 호출하는 것보다  
       거의 항상 느리다.  
    - 즉, auto 접근방식보다 메모리와 시간을 더 많이 소비하고, 때에 따라서는 메모리 부족(out-of-memory) 예외를 유발할 수 있다.

## auto를 사용할 경우 장점
  - 지금까지 살펴본 auto의 장점은 아래와 같다.
    - 변수 초기화 누락을 방지하고, 
    - 장황한 변수 선언을 피하는 것, 
    - 클로저를 직접 담는 것
  
  - 이것들 말고도 다른 장점이 하나 더 있다.  
    - '형식 단축(type shortcut)'이라고 부르는 것과 관련된 문제를 피할 수 있다.  
    - 아래의 예제를 보자.  
      v.size()의 return type은 std::vector<int>::size_type인데, 
      이점을 아는 개발자들은 별로 없어서 보통 unsigned int 변수에 size()의 return값을 담고는 한다.  
      그래서 부호없는 정수이므로 unsigned키워드만으로 충분하다고 생각해서 아래처럼 선언하는 경우가 종종 있다.  
      그러나 32비트 Windows에서는 unsigned와 std::vector<int>::size_type이 같은 크기이지만,
      64비트 Windows에서는 unsigned가 32비트인 반면 std::vector<int>::size_type은 64비트이다.  
      이는 32비트 Windows에서는 잘 작동하지만, 64비트 Windows에서는 오작동할 수 있음을 의미한다.  
      auto를 사용하면 형식연역을 통해서 size_type으로 초기화되므로 이런 시간낭비를 하지 않아도 된다.  

~~~C++
std::vector<int> v;
...
unsigend sz = v.size(); // v.size()의 return value가 부호없는 정수라고 생각해서 이런 선언을 할 경우,
                        // 32 bit Windows에서는 잘 동작하지만, 
                        // 64 bit Windows에서는 왼값과 오른 값의 형식이 달라서 오동작을 할 가능성이 있다. 

auto sz = v.size();   // sz의 형식이 auto의 형식연역을 통해서 std::vector<int>::size_type으로 선언된다.
~~~

 - 또 하나의 장점을 살펴보자.  
~~~C++
std::unordered_map<std::string, int> m;
...

for (const std::pair<std::string, int>& p : m)
{
  ...  // p로 뭔가 수행.
}
~~~
    - std::unordered_map의 key부분이 const라는 점을 주목하자.  
      - 키 부분이 const이므로, 해시테이블(std::unordered_map)에 담긴 std::pair의 형식은  
        std::pair<std::string, int>가 아니라 std::pair<const std::string, int>이다.  
      - 그런데 이는 루프 위의 변수 p에 대해 선언된 형식과는 다르다.  
      그래서 컴파일러는 std::pair<const std::string, int> 객체들을 어떻게든  
      std::pair<std::string, int> 객체들(p의 선언된 형식에 해당하는)로 변환하려 든다.  
      루프의 각 반복에서, 컴파일러는 p를 묶고자 하는 형식의 임시 객체를 생성하고,  
      m의 각 객체를 복사하고, 참조 p를 그 임시 객체에 묶음으로써 그러한 변환을 실제로 수행한다.  
      그 임시 객체는 루프 반복의 끝에서 파괴된다.  
      아마 이 코드를 작성한 사람의 의도는 그냥 참조 p를 m의 각 요소에 묶는 것이겠지만,  
      놀랍게도 루프 내부적으로는 상당히 복잡한 작업이 일어난다.  
      이러한 의도하지 않은 형식 불일치 역시 auto로 날려버릴 수 있다.  

~~~C++
for (const auto& p : m)
{
  ...  // p로 뭔가 수행.
}
~~~

auto는 필수가 아니라 선택이다.  
명시적 형식 선언을 사용하는 것이 더 깔끔하거나 유지보수하기 쉬운,  
또 다른 어떤 방식으로 더 나은 코드로 이어진다는 결론을 내렸다면,  
명시적 선언을 계속 사용하면된다.

auto를 사용하지 않으면 어떤 함수의 반환 타입을 int에서 long으로 변경할 경우, 호출 지점을 찾아 일일히 수정해주어야 하지만, auto를 사용하면 일일히 고쳐야 할 번거로움이 사라진다.  
여기까지가 저자의 생각이다.  

결과적으로, 저자의 결론은 auto를 사용하는 것이 훨씬 좋다는 것인데..  
내 생각은 조금 다르다. auto를 남발할 경우, 코드의 추적이 복잡해진다. 역설적으로 유지보수가 어려워진다.  
예제처럼 UI와 관련된 프로그램일 경우, 디버깅이 쉬운 편이지만,  
Engine이나 백그라운드 서비스를 개발할 경우, 그리고 코드 규모가 큰 프로젝트를 개발할 경우, 디버깅 시에 디버거를 쓰지 못하는 경우가 있다.  
Linux환경에서는 더더욱 그렇다. 이럴 경우, 코드 리뷰를 꼼꼼히 해야하는데, 이때 auto의 지옥에서 빠져나오지 못할 것이다.  
보통 a라는 함수를 부르는 부분을 추적하기위해서 a, b, c, d, e,,.. 더 depth를 깊게 들어가는 경우가 있을 텐데,
type이 auto로 되어있네?... 아.. 다시 쭉 올라가서 타입을 보자.. 하는 과정이 반복될 것이다.  