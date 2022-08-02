## 항목 2: auto의 형식 연역 규칙을 숙지하라
auto에 사용되는 형식 연역 규칙들은 항목 1에서의 형식 연역 규칙들과 같다.  
auto 형식 연역이 곧 템플릿 형식 연역이기 때문이다.  
템플릿 형식 연역과 auto 형식 연역 사이에는 직접적인 대응 관계가 존재한다.  
 그 둘을 문자 그대로 알고리즘 적으로 상호변환할 수 있다.  
항목 1에서는 다음과 같은 일반적인 함수템플릿을 일반적인 호출을 예로 들어서 템플릿 형식 연역을 설명했다.  
~~~C++
template<typename T>
void func(ParamType param); // 일반적인 함수 템플릿

func(expr);                 // 어떤 표현식(expr)으로 func를 호출
// func 호출에서 컴파일러는 expr을 이용해서 T의 형식과 ParamType의 형식을 연역한다.
~~~
  
auto를 이용해서 변수를 선언할 때 auto는 템플릿의 T와 동일한 역할을 하며,  
변수의 형식 지정자(type specifier)는 ParamType과 동일한 역할을 한다.  
  
~~~C++
auto x = 27;        // x의 형식 지정자는 그냥 auto 자체이다.
const auto cx = x;  // 형식 지정자는 const auto이다.
const auto& rx = x; // 형식 지정자는 const auto&이다.
~~~

위의 변수 선언을 템플릿 함수의 연역방식으로 살펴보자.
~~~C++
template<typename T>              // x의 형식을 연역하기 위한
void func_for_x(T param);         // 개념적인 템플릿

func_for_x(27);                   // 개념적인 호출: param에 대해
                                  // 연역된 형식이 바로 x의 형식이다.

template<typename T>              // cx의 형식을 연역하기 위한
void func_for_cx(const T param);  // 개념적인 템플릿

func_for_cx(x);                   // 개념적인 호출: param에 대해
                                  // 연역된 형식이 바로 cx의 형식이다.

template<typename T>              // rx의 형식을 연역하기 위한
void func_for_rx(const T& param); // 개념적인 템플릿

func_for_rx(x);                   // 개념적인 호출: param에 대해
                                  // 연역된 형식이 바로 rx의 형식이다.
~~~

즉, auto의 형식 연역은 템플릿 형식 연역과 동일한 것을 알 수 있다.
(한가지 예외(initializer_list)를 제외하면 말이다.)

auto의 형식 연역 역시 템플릿 형식연역과 동일하게 다음 세가지 경우로 나뉜다.
 - 경우 1: 형식 지정자가 포인터나 참조 형식이지만 보편 참조(universal reference)는 아닌 경우.
 - 경우 2: 형식 지정자가 보편 참조인 경우
 - 경우 3: 형식 지정자가 포인터도 아니고 참조도 아닌 경우
~~~C++
// 경우 1, 경우 3
auto x = 27;        // 경우 3 (x는 포인터도 아니고 참조도 아님)
const auto cx = x;  // 경우 3 (cx 역시 둘 다 아님)
const auto& rx = x; // 경우 1 (rx는 보편 참조가 아닌 참조)

// 경우 2
auto&& uref1 = x;   // x는 int이자 왼값이므로
                    // uref1의 형식은 int &
auto&& uref2 = cx;  // cx는 const int이자 왼값이므로
                    // uref2의 형식은 const int &
auto&& uref3 = 27;  // 27은 int이자 오른값이므로
                    // uref3의 형식은 int&&
~~~
---------------------------------------------------------------------
### 템플릿 형식 연역에 비해 한가지 다른점 - initializer_list 형식 연역
C++98에서는 다음 두가지 구문이 가능했다.
~~~C++
int x1 = 27;
int x2(27);
// 균일 초기화(uniform initialization)를 지원하는 C++11에서는 위의 두 구문과 더불어
// 다음과 같은 구문들을 사용할 수도 있다.
int x3 = {27};
int x4{27};
~~~
총 네 가지 구문이 존재하나, 결과적으로 값이 27인 int가 생긴다는 점은 모두 동일하다.
위의 구문을 auto를 사용해서 선언한 코드를 살펴보자
~~~C++
auto x1 = 27;   // 형식은 int, 값은 27
auto x2(27);    // (동일) 형식은 int, 값은 27
auto x3 = {27}; // 형식은 std::intializer_list<int>, 값은 {27}
auto x4{27};    // (동일)
~~~
이 선언들은 문제었이 컴파일되지만, 조금 다르다.  
x1, x2의 경우 형식이 int이고 값이 27인 변수를 선언하지만,  
x3, x4의 경우는 값이 27인 원소 하나를 담은 std::initializer_list<int> 형식의 변수를 선언한다.  
이는 auto의 형식 연역 규칙 때문이다. 중괄호 쌍으로 감싸인 형태로 초기화된 변수는 std::initializer_list 형식으로 연역된다.  
만약 중괄호 초기치의 값들이 서로 다르면 형식연역을 할 수 없으면 컴파일이 거부된다.

~~~C++
auto x5 = {1, 2, 3.0} // 오류! std::intialiizer_list<T>의 T를 연역할 수 없음
~~~
중괄호 초기치를 구성하는 값들의 형식이 한 종류가 아니라서 그 형식 연역이 실패한다.
연역과정을 풀어서 설명하면, 다음과 같다.  

~~~C++
x5는 std::initializer_list로 연역된다. -> 이 템플릿의 인스턴스는 'T에 대한 std::initializer_list<T>'이다  
선언을 완성하려면 반드시 T의 형식을 연역해야 한다.  
그런데, T의 형식이 한가지가 아니기 때문에 형식연역이 되지 않는다.
~~~~
auto 형식연역과 템플릿 형식연역은 이처럼 중괄호 초기치가 관여할 때에만 차이를 보인다.  
  
auto의 경우는 중괄호 초기치를 initializer_list로 연역하지만,  
템플릿 함수에 중괄호 초기치를 전달하면 형식 연역이 실패해서 컴파일이 거부된다.  

~~~C++
auto x = {11, 23, 9};   // x의 형식은 std::intializer_list<int>

template<typename T>    // x의 선언에 해당하는 매개변수
void func(T param);     //  선언을 가진 템플릿

func({11, 23, 9});      // 오류! T에 대한 형식을 연역할 수 없음.
~~~

paramType을 std::initializer_list<T>로 변경하면 형식이 제대로 연역된다.
~~~C++
template<typename T>
void func(std::intializer_list<T> initList);

func({11, 23, 9});      // T는 int로 연역되며, initList의 형식은
                        //  std::initializer_list<int>로 연역된다.
~~~
즉, auto 형식 연역과 템플릿 연역의 실질적인 차이는,  
auto는 중괄호 초기치가 std::initializer_list를 나타낸다고 가정하지만  
템플릿 형식연역은 그렇지 않다는 것 뿐이다.  

-------------------------------------------------------------------
### auto에 대해 C++14에서 알아두면 좋을 것
C++14에서는 함수의 반환 형식을 auto로 지정해서 컴파일러가 연역하게 만들 수 있다.  
람다의 매개변수 선언에 auto를 사용하는 것도 가능하다.  
그러나 그러한 용법들에는 auto형식 연역이 아니라 템플릿 형식 연역 규칙이 적용된다.  
--> 중괄호 초기치를 돌려주는 함수의 반환 형식을 auto로 지정하면 컴파일이 실패한다.
~~~C++
auto createInitList()
{
  return {1, 2, 3};     // 오류! {1, 2, 3} 의 형식을 연역할 수 없음.
}

std::vector<int> v;

auto resetV = [&v](const auto& newValue) { v = newValue; }; // C++14

resetV({1, 2, 3});    // 오류! {1, 2, 3}의 형식을 연역할 수 없음.
~~~
