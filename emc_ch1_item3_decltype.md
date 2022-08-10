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
컨테이너, 색인 하나를 받고 사용자를 인증한 후 대괄호 색인화를 통해서 컨테이너의 한 요소를 돌려주는 함수
를 작성할 경우,  함수의 반환 형식은 색인화 연산의 반환 형식과 동일해야 한다.
대체로, 형식 T의 객체들을 담은 컨테이너에 대한 operator[]연산은 T&을 돌려준다.
std::deque, std::vector 도 그렇다.
operator[]의 반환 형식이 컨테이너에 따라 다를 수 있다.
~~~C++
template<typename Container, typename Index>
auto authAndAccess(Container& c, Index i)
    -> decltype(c[i])
    {
        
    }
~~~

