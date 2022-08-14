# 항목 3: decltype의 작동 방식을 숙지하라

대부분의 경우 decltype은 독자가 예측한 그 형식을 말해준다.  
아래의 예를 보면, 템플릿과 auto의 형식 연역과는 달리 decltype은 이름이나 표현식의 구체적인 형식을 그대로 말해준다.  
하지만 가끔은 예상 밖의 결과가 나온다.(조금 이따가 설명할 것이다.)  
~~~C++
const int i = 0;            // decltype(i)는 const int

bool f(const Widget& w);    // decltype(w)는 const Widget&
                            // decltype(f)는 bool(const Widget&)

struct Point {
    int x, y;               // decltype(Point::x)는 int
};                          // decltype(Point::y)는 int

Widget w;                   // decltype(w)는 Widget

if (f(w)) ...               // decltyp(f(w))는 bool

template<typename T>        // std::vector를 단순화한 버전
class vector {
public:
    ...
    T& operator[](std::size_t index);
    ...
};

vector<int> v;              // decltype(v)는 vector<int>
...
if (v[0] == 0) ...          // decltype(v[0])은 int&

~~~

C++11에서 decltype은 함수의 반환 형식이 그 매개변수 형식들에 의존하는 함수 템플릿을 선언할 때 주로 쓰인다.  
컨테이너, 색인 하나를 받고 대괄호 색인화를 통해서 컨테이너의 한 요소를 돌려주는 함수를 예로 살펴보자.  
아래처럼 형식 T의 객체들을 담은 컨테이너에 대한 operator[]연산은 T&을 돌려주는 함수가 될 것이다.  
std::deque, std::vector처럼, operator[]의 반환 형식이 컨테이너에 따라 다를 수 있다.  
이런 함수를 구현할 경우 아래처럼 decltype을 이용해서 반환형식을 손쉽게 표현할 수 있다.  
(decltype을 쓰지 않으면 int, bool, double 등의 타입별로 다 오버로딩해야하므로..)  
~~~C++
template<typename Container, typename Index>  // 작동하지만, 
auto authAndAccess(Container& c, Index i)     // 좀 더 정련할 
    -> decltype(c[i])                         // 필요가 있다
{
    authenticateUser(); // 함수와 관련된 인증 동작이 있다고 가정, 주제와는 관련이 없는 별 의미없는 코드
    return c[i];
}
~~~
  
함수 이름 앞에 auto를 지정하는 것은 C++11의 후행 반환 형식(trailing return type) 구문이 쓰인다는 것을 의미한다.  
즉, 함수의 반환 형식을 이 위치(함수의 앞부분)가 아니라 매개변수 목록 다음에 선언하겠다는 점을 나타낸다.  
이런 방식은 반환형식 반환 형식을 매개변수들을 이용해서 지정할 수 있다는 장점이 있다.  
상기의 예에서는 매개변수 c와 i를 이용해서 반환 형식을 지정했다. 통상적인 방법이라면 c와 i는 둘 다 아직 선언되지 않았으므로 사용할 수 없다.  
  
C++11은 람다 함수가 한 문장으로 이루어져 있다면 그 반환 형식의 연역을 허용하며,  
C++14는 허용 범위를 더욱 확장해서 모든 람다와 모든 함수의 반환 형식 연역을 허용한다.  
그래서 C++14를 사용할 경우, 후행 반환형식을 생략이 가능하다.
  
## 템플릿 형식 연역 과정에서 참조성이 무시되는 문제 -> decltype(auto)로 해결
------------------------------------------------------------------------------
### CASE 1: 후행 반환 형식
항목 2에서 봤듯이 함수의 반환 형식에 auto가 지정되어 있으면 컴파일러는 템플릿 형식연역을 적용한다.  
대부분의 T객체들을 담은 컨테이너에 대한 operator[]연산은 T&을 돌려주지만,  
문제는 템플릿 형식 연역 과정에서 초기화 표현식의 참조성이 무시된다는 점인데, 아래의 예제코드를 살퍄보자.  
~~~C++
std::deque<int> d;
...
authAndAccess(d, 5) = 10; //사용자를 인증하고, d[5]를 돌려주고,
                          // 그런 다음 10을 d[5]에 배정한다.
                          // 이 코드는 컴파일되지 않는다!
~~~
  
여기서 d[5]는 int&를 돌려주나, authAndAccess에 대한 auto 반환 형식 연역 과정에서  
참조가 제거되기 때문에 결국 반환 형식은 int가 된다.  
함수의 반환값으로서의 이 int는 오른값이며, 즉, 위의 예제코드는 오른 값 int에 10을 배정하려하고 있다.  
오른 값(임시 값)에 값을 배정하는 것은 C++에서 금지되어 있어서 컴파일되지 않게 된다.  
  
이 문제를 해결하기 위해서는 decltype 형식 연역이 적용되어서  
authAndAccess가 c[i]의 반환 형식과 정확히 동일한 형식을 반환하게 만들어야한다.  
이런 경우 C++14의 decltype(auto)지정자를 통해서 그러한 일이 가능하다.  
auto는 해당 형식이 연역되어야 함을 뜻하고, decltype은 그 연역 과정에서 decltype 형식 연역 규칙들이 적용되어야 함을 의미한다.  
아래 예제코드 처럼 적용이 가능하다.  
~~~C++
template<typename Container, typename Index> // C++14
decltype(auto)                                // 작동하지만, 좀 더 정련할 필요가 있다.
authAndAccess(Container& c, Index i)
{
  authenticateUser();
  return c[i]
}
~~~
  
### CASE 2: 변수 초기화 시에...
decltype(auto)는 함수 반환 형식에만 사용할 수 있는 것이 아니다. 변수를 선언하거나 초기화 표현식에도 decltype형식 연역 규칙을 적용할 수 있다.  
~~~C++
Widget w;
const Widget& cw = w;
auto myWidget1 = cw;              // auto 형식 연역: myWidget1의 형식은 Widget
decltype(auto) myWidget2 = cw;    // decltype 형식 연역: myWidget2의 형식은 const Widget&
~~~


## decltype(auto)를 사용한 방법을 개선해보자.
--------------------------------------------------------------------
다시 C++14의 decltype(auto)를 사용해서 함수를 정의한 부분으로 돌아가서 더 정련할 수 있는 방법을 보자.  
컨테이너 c는 non-const 객체에 대한 왼값 참조로서 함수에 전달되고 있는데, 이 때문에 함수에 오른값 컨테이너는 전달할 수 없다는 것이 문제가 된다. (오른 값을 왼값 참조에 묶을 수 없기 때문이다. const에 대한 왼값 참조에는 묶을 수 있지만.. )  

authAndAccess에 오른값 컨테이너를 넘기는 것은 극단적인 경우(edge case)이긴 하다.  
임시 객체로서의 오른값 컨테이너는 일반적으로 authAndAccess 호출을 담은 문장의 끝에서 파괴되므로
그 컨테이너 안의 한 요소를 지정하는 참조는 그 요소를 생성하는 문장의 끝에서 지칭 대상을 잃게 된다.  
그렇긴 하지만, authAndAccess에 임시 객체를 넘겨줄 수 있게 만드는 것은 여전히 합당하다.  
아래 예제처럼 클라이언트가 그냥 임시 컨테이너의 한 요소의 복사본을 만들고 싶을 수도 있기 때문이다.  
~~~C++
std::deque<std::string> makeStringDeque();    // 팩터리 함수
// makeStringDeque가 돌려준 deque의
// 다섯 번째 원소의 복사본을 만든다.
auto s = authAndAccess(makeStringDeque(), 5);
~~~
  
이런 용법을 지원하려면 authAndAccess가 왼값 뿐만 아니라 오른값도 받아들이도록 선언을 고쳐야한다.  
중복적재를 사용할 수도 있지만(왼값 참조 매개변수를 받는 중복적재 버전과 오른값 참조 매개변수를 받는 중복적재 버전을 따로 만들어서), 그러면 관리해야 할 함수가 두개가 된다.  
이를 피하는 한 가지 방법은 왼 값과 오른 값 모두에 묶일 수 있는 참조 매개변수를 authAndAccess에 도입하는 것이다.  
항목 24에서 설명하겠지만, 보편 참조가 바로 그런 용도로 쓰인다.  
~~~C++
template<typename Container, typename Index>          // 이번에는 c가 보편 참조
decltype(auto) authAndAccess(Container&& c, Index i);
~~~
  
지금 시점에서 이 템플릿이 다루는 컨테이너의 형식과 컨테이너의 한 요소에 접근하는데 쓰이는 객체의 형식은 알 수 없다.  
미지의 형식의 객체에 대해 값 전달 방식을 적용하면 불필요한 복사 때문에 성능이 하락하거나,  
객체가 잘려서(항목 41 참고) 프로그램의 행동에 문제가 생기거나, 또는 동료 개발자의 비웃음을 살 위험이 있다.  
그렇긴 하지만, 컨테이너 색인에 대해서는 표준 라이브러리가 색인 값을 사용하는 방식(ex: std::string, std::vector, std:: deque의 operator[]에 쓰이는)을 따르는 것이 합당해보인다.  
그래서 색인 매개변수에 대해서는 값 전달방식을 고수하기로 하자.  
  
  
## std::forward 적용
--------------------------------------------------------------
변화된 선언에 맞게 템플릿의 구현도 고칠 필요가 있다. 구체적으로 항목 25의 조언에 따라 다음과 같이 보편 참조에 std::forward를 적용하기로 하자.  
~~~C++
template<typename Container, typename Index>
decltype(auto)
authAndAccess(Container&& c, Index i)
{
  authenticateUser();
  return std::forward<Container>(c)[i]; // std::forward 사용
}
~~~
  
C++11을 사용하는 코드는 아래와 같다.  
~~~C++
template<typename Container, typename Index> // 최종 C++11번전
auto
authAndAccess(Container&& c, Index i)
  -> decltype(std::forward<Container>(c)[i]) // std::forward 사용
{
  authenticateUser();
  return std::forward<Container>(c)[i];l
}
~~~

## decltype 사용시에 주의할 점
------------------------------------------------------------------
이름보다 복잡한 왼값 표현식에 대해서는 일반적으로 decltype이 항상 왼값 참조를 보고한다.  
즉, 이름이 아닌, 그리고 형식이 T인 어떤 왼값 표현식에 대해 decltype은 T&을 보고한다.  
어차피 대부분의 왼값 표현식에는 태생적으로 왼값 참조가 포함되어 있으므로, 이것 때문에 뭔가가 달라지는 경우는 드물다.  
그러나 이런 작동 방식이 뭔가 차이를 만드는 경우도 있다. 아래 예제 코드를 보자.  
  
아래 코드에서 x는 변수의 이름이므로 decltype(x)는 int이다.  
그러나 x를 괄호로 감싸서 "(x)"를 만들면 이름보다 복잡한 표현식이 된다.  
이름으로서의 x는 하나의 왼값이며, C++은 (x)라는 표현식도 왼값으로 정의한다.  
따라서 decltype((x))는 int&이다. 이름을 괄호로 감싸면 decltype이 보고하는 형식이 달라지는 것이다!  
~~~C++
int x = 0;
~~~
  
C++11에서는 이것이 그냥 드물게 만나는 신기한 현상 정도이다.  
그렇지만 decltype(auto)를 지원하는 C++14에서는 return문 작성 습관의 사소한(겉으로 보기에) 차이 때문에  
함수의 반환 형식 연역 결과가 달라지는 사태가 벌어질 수 있다.  
~~~C++
decltype(auto) f1() 
{
  int x = 0;
  ...
  return x;   // decltype(x) 는 int이므로 f1은 int를 반환
}

decltype(auto) f2()
{
  int x = 0;
  ...
  return (x); // decltype((x))는 int&이므로 f2는 int&을 반환
}
~~~
  
f2가 f1과는 다른 형식을 돌려준다는 점 뿐만 아니라, 자신의 지역 변수에 대한 참조를 돌려준다는 점도 주목하자.  
이런 종류의 코드를 작성하는 것은 미정의 행동으로 직행하는 특급 열차에 올라타는 것과 같다.  
이 예의 주된 교훈은, decltype(auto)는 아주 조심해서 사용해야 한다는 것이다.  
형식연역 표현식에서 겉으로 보기에는 중요하지 않은 세부사항이라도 decltype(auto)가 보고하는 형식에 영향을 미칠 수 있다.  
애초에 예상했던 형식이 실제로 연역되었는지 확인하려면 항목 4에서 설명하는 기법들을 사용하면 된다.  
  
decltype이(그 자체로 사용하든 auto와 함께 사용하든) 아주 가끔 뜻밖의 형식을 연역하기도 하지만, 그것은 정상적인 상황이 아니다. 보통의 경우 decltype(declared type)은 기대한 바로 그 형식을 산출한다. 