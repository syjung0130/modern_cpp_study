## 항목1: 템플릿 형식 연역 규칙
예제 코드를 먼저 보면서 템플릿 형식 연역규칙을 보자.  
함수템플릿의 선언과 호출은 아래 코드와 같이 이루어진다.  
 - 함수템플릿의 선언  
~~~C++
template<typename T>
void func(ParamType param);
~~~
  
 - 이 함수를 호출하는 코드  
~~~C++
func(expr);
~~~
  
컴파일 도중에 컴파일러는 전달받은 expr을 이용해서 두 가지 형식을 연역한다.  
 - T형식에 대한 연역(expr -> T형식)
 - ParamType에 대한 연역(T -> ParamType형식)
  
T형식과 ParamType에 대한 형식이 다른 경우가 많은데, ParamType에 const나 참조 한정사(&, &&)같은 수식어들이 붙는 경우가 있기 때문이다.  
  
ParamType에 수식어들이 붙는 경우 중 const가 붙는 아래의 예제 코드를 보자.  
~~~C++
//템플릿 함수 선언
template<typename T>
void func(const T& param);//ParamType은 const T&

//함수 호출부
int x = 0;
func(x);//int로 func를 호출
~~~
  
이 경우, (expr는 int)
 - T는 int로 연역되고  
 - ParamType은 const int&으로 연역된다.  
T에 대해 연역된 형식(결과)이 함수에 전달된 인수의 형식(expr)과 같을 것이라고,  
즉, T가 expr의 형식이라고 기대할 수 있다. 위의 예에서는 그렇기는 하지만, 항상 그렇지는 않다.  
T에 대해 연역된 형식(결과)은 expr의 형식에 의존할 뿐만 아니라 ParamType의 형태에도 의존하는데, 
ParamType의 형태에 따라 연역되는 형태는 총 세 가지 경우(시나리오)로 나뉜다.  
 - ParamType이 포인터 또는 참조형식(보편 참조는 제외 - 항목24에서 참고)인 경우  
 - ParamType이 보편 참조인 경우  
 - ParamType이 포인터도 아니고 참조도 아닌 경우  

아래의 예제코드를 기반으로 세가지 경우를 살펴보자.  
~~~C++
template<typename T>
void func(ParamType param);

func(expr);//expr로부터 T와 ParamType을 연역
~~~
  
### 경우1: ParamType이 포인터 또는 참조형식이지만 보편참조는 아님
형식 연역이 다음과 같이 진행된다.  
~~~
 1. 만일 expr이 참조 형식이면 참조 부분을 무시한다.
 2. 그런 다음 expr의 형식을 ParamType에 대해 패턴 부합(pattern-matching) 방식으로 대응시켜서 T의 형식을 결정한다.  
~~~
아래 예제코드로 살펴보자.  
~~~C++
//함수템플릿 선언
template<typename T>
void func(T& param);    // param은 참조 형식

//변수 선언
int x = 27;             // x는 int
const int cx = x;       // cx는 const int
const int& rx = x;      // rx는 const int인 x에 대한 참조

//함수 호출
f(x);                   // T는 int, param의 형식은 int &

f(cx);                  // T는 const int,
                        // param의 형식은 const int&

f(rx);                  // T는 const int,
                        // param의 형식은 const int&
~~~  
 - 둘째, 셋째 호출에서 cx와 rx에 const값이 배정되었기 때문에 T가 const int로 연역되었다.  
 그래서 매개변수 param의 형식은 const int&가 되었다.  
 - const 객체를 전달하는 호출자는 객체가 수정되지 않을 것이라 기대한다. (해당 매개 변수가 const에 대한 참조일 것이라 기대한다..)  
 이처럼, 객체의 const성(const-ness; 상수성)은 T에 대해 연역된 형식(T)의 일부가 되기 때문에  
 T& 매개변수를 받는 템플릿에 const객체를 전달해도 안전하다. --> expr에 const가 포함되어도 T의 일부가 된다.  
 - 셋째 호출에서, 형식 연역 과정에서 rx의 참조성이 무시되기 때문에 T가 비참조로 연역되었다.(rx의 형식은 참조이지만.) --> expr에 참조가 포함되면 T로 연역되는 과정에서 무시된다(제거된다).  

param이 참조가 아니라 포인터(또는 const를 가리키는 포인터)라도 형식 연역은 본질적으로 같은 방식으로 진행된다.  
~~~C++
template<typename T>
void func(T* param);    // 이번에는 param이 포인터

int x = 27;             // x는 int
const int *px = &x;     // px는 const int로서의 x를 가리키는 포인터

func(&x);               // T는 int, param의 형식은 int*
func(px);               // param의 형식은 const int*
~~~

### 경우2: ParamType이 보편참조임
템플릿이 보편 참조 매개변수를 받는 경우는 조금 어렵다.  
매개변수의 선언은 오른값 참조와 같은 모습이지만  
(즉, 형식 매개변수 T를 받는 함수 템플릿에서 보편 참조의 선언 형식은 T&&이다.)  
왼값 인수가 전달되면 오른값 참조와는 다른 방식으로 행동한다.  
자세한 내용은 항목24 참고. 결론만 보자면 이렿다.  
 - 만일 expr이 왼값이면, T와 ParamType 둘 다 왼값참조로 연역된다. 이는 이중으로 비정상적인 상황이다.  
 첫째로, 템플릿 형식 연역에서 T가 참조형식으로 연역되는 경우는 이것이 유일하다.  
 둘째로, ParamType의 선언구문은 오른값 참조와 같은 모습이지만, 연역된 형식은 왼값 참조이다.  
 - 만일 expr이 오른값이면, '정상적인'(즉 경우 1의) 규칙들이 적용된다.  
  
~~~C++
template<typename T>
void func(T&& param);   //이번에는 param이 보편 참조

int x = 27;             // x는 int
const int cx = x;       // cx는 const int
const int& rx = x;      // rx는 const int인 x에 대한 참조

func(x);                // x는 왼값, 따라서 T는 int&,
                        // param의 형식 역시 int&
func(cx);               // cx는 왼값, 따라서 T는 const int&,
                        // param의 형식 역시 const int&
func(rx);               // rx는 왼값, 따라서
~~~
  
  보편참조의 경우, 이렇게 연역되는 구체적인 이유는 항목24를 참고하자.  
  일단 지금은, 보편 참조 매개변수에 대한 형식 연역 규칙들이 왼값 참조나 오른값 참조 매개변수들에 대한 규칙들과는 다르다는 점만 기억하자.  
  보편 참조가 관여하는 경우에는 왼값인수와 오른값인수에 대해 다른 연역 규칙들이 적용된다.  
  
### 경우3: 0
  