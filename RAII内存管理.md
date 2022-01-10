# RAII内存管理

### C++思想

#### 封装

将多个逻辑上相关联的变量包装成一个类

**封装不变性：** 常遇到当修改一个成员的时候，其他成员也需要被修改，否则出错，这种情况出现时，意味着你需要把成员变量的读写封装成**成员函数**。

不要滥用get，set封装，仅当“修改一个成员的时候，其他成员也需要被修改”时使用。

#### RAII（Resource Acquisition Is Initialization）

资源获取视为初始化，反之，资源释放视为销毁。C++除了用于初始化的**构造函数（constructor）**,还有用于销毁的**解构函数（destructor）。**

#### 解构函数

如果没有解构函数，则每个带有返回的分支都要手动释放所有之前的资源，与Java，Python等立即回收语言不同，C++的解构函数是显式的，离开作用域自动销毁，毫不含糊。

#### 构造函数

**初始化表达式**

```cpp
struct Pig{
    std::string m_name;
    int m_weight;
    Pig() : m_name("Peki"),m_weight(80)
    {}
    Pig(string name, int weight) : m_name(name),m_weight(weight)
    {}
    explicit Pig(int weight):m_name("重"+std::to_string(weight)+"kg的猪")
        					,m_weight(weight)
    {}
};

int main()
{
    Pig pig("佩奇"，80)；
    Pig pig{"佩奇"，80}；
        
}
```

为什么要用初始化表达式？

1. 假如类成员为const和引用
2. 假如类成员没有无参数构造函数
3. 避免重复初始化。更高效

初始化表达式是直接将数据写入内存空间，若在函数体内赋值的话，是先开辟内存空间，再进行赋值操作，会更慢一些。

**explicit**

显式的，再构造函数前加上，表明不允许隐式调用该构造函数，即

```cpp
Pig pig=80;//编译不过
Pig pig(80);//编译通过
```

explicit对多个参数也起作用！

多个参数时，explicit的作用体现在禁止从一个{}表达式初始化。

如果你需要在一个返回Pig的函数里用：

```cpp
return{"佩奇"，80};
```

的话，就不要加explicit。

**使用{}和()调用构造函数，有什么区别？**

1. ()有强制类型转换，{}没有，即int(3.14f)不会报错，int{3.14f}会。
2. 可读性，Pig(1.2)看上去像个函数，Pig{1.2}更明确。

Google Style 中明确提出别再通过()调用构造函数，需要类型转换的时候用

```cpp
static_cast<int>(3.14f)//代替int(3.14f)
reinterpret_cast<void*>(0xb8000)//而不是(void*)0xb8000
```

**POD(plain-old-data)陷阱**

除了自定义构造函数外，编译器还会自动生成构造函数。

当一个类没有定义任何构造函数，且每个成员都有无参构造函数时，编译器会自动生成一个无参构造函数Pig()，他会调用每个成员的无参构造函数。

**notice：**这些类型**不会被初始化为0：**

1. int,float,double等基础类型
2. void\*,Object*等指针类型
3. 完全由这些类型组成的类

这些类型被称为**POD**(plain-old-data)。这么做是出于兼容性和性能考虑。

```cpp
int x{};
void *p{};
//与
int x{0};
void *p{nullptr};
//等价，内存都会清0，但是若不写{}，他的值就是随机的值。
```

#### 编译器自动生成的函数：全家桶

```cpp
struct C{
    c();						//默认构造函数
    
    C(C const &c);				//拷贝构造函数
    C(c &&c);					//移动构造函数
    C &operator=(C const &c);	//拷贝赋值函数
    C &operator=(C &&c);		//移动赋值函数
    
    ~C();						//解构函数
}

int main()
{
    C c1 = c2;					//拷贝构造
    C c1 = std::move(c2);		//移动构造
    						
   	c1 = c2;					//拷贝赋值
    c1 = std::move(c2);			//移动赋值
    
    C c1 = C();					//移动构造
    c1 = C();					//移动赋值
    return c2;					//移动赋值
}
```



#### 三五法则

若是自定义解构函数，一定要自定义拷贝构造函数和拷贝赋值函数，若要提高效率，最好还要自定义移动构造函数和移动赋值函数。

**拷贝构造函数**

默认是浅拷贝，若成员变量中有指针，它会直接复制指针值，而不是复制所指内容。如果这时候在析构函数里面有释放指针的行为，则容易导致double free，所以自定义析构函数后要定义拷贝构造函数。

**拷贝赋值函数**

区分两种拷贝可以提高性能。

```cpp
int x=1;//拷贝构造
x=2;	//拷贝赋值
```

**拷贝赋值函数=解构函数+拷贝构造函数**

拷贝构造：直接在未初始化的内存上依据他构造我

拷贝赋值：先销毁现有的我，再重新构造他

内存的销毁和重新分配可以通过realloc，从而就地利用当前现有的m_data,避免重新分配。

```cpp
Vector &operator=(Vector const &other){
	m_size=other.m_size;
    m_data=(int*)realloc(m_data,m_size*sizeof(int));
    memcpy(m_data,other.m_data,m_size*sizeof(int));
    return *this;
}
```



若是对提高性能不感兴趣，可以这么写：

```cpp
Vector &operator=(Vector const &other){
    this->~Vector();
    new(this) Vector(other);
    return *this;
}
```



#### 区分移动和拷贝

有时候需要把一个对象v2移动到v1上，而不需要涉及实际数据的拷贝。

时间复杂度：移动为O（1），拷贝为O（n）。

我们可以用std::move实现移动

v2被移动到v1后，原来的v2也会被清零。

**移动进阶：交换，std::swap：**在高性能计算中用来实现双缓存（ping-pong buffer），时间复杂度同样为O（1）。

**哪些情况会触发移动**

```cpp
return v2;					//v2做返回值
v1=std::vector<int>(200);·	//就地构造v2
v1=std::move(v2)			//显示移动
```

**哪些情况会触发拷贝**

```cpp
return std::as_const(v2);	//显示拷贝
v1=v2;						//默认拷贝
```

**注意:**std::move 和 std::as_const 只是负责类型转换，实际产生移动，拷贝效果的是在类的构造、赋值函数里面。



**移动函数**

默认移动构造=拷贝构造+他解构+他默认构造

默认移动赋值=拷贝赋值+他解构+他默认构造

```cpp
Vector(Vector &&other){
    m_size=other.m_size;
    other.m_size=0;
    m_data=other.m_data;
    other.m_data=nullptr;
}
```

若对于性能不敏感可以理解为

移动赋值=我解构+移动构造

```cpp
Vector &operator=(Vector &&other){
    this->~Vector();
    new(this)Vector(std::move(other));
    return *this;
}
```

**小技巧：如果有移动赋值函数，可以删除拷贝赋值函数**

此时当调用：

```cpp
v2=v1;
```

因为拷贝赋值被删了，编译器会尝试：

```cpp
v2=List(v1)；
```

从而变成先就地调用拷贝构造函数，再调用移动赋值函数。

### 自制Vector

#### C语言的内存管理

c语言在使用堆上内存时要用malloc申请，用free释放，十分麻烦。我们可以把这一过程封装，就有了STL容器。

如下的例子，看看我们是怎样通过一步步封装，实现Vector

```c
#include <stdlib.h>
#include <stdio.h>

int main() {
    size_t nv = 2;
    int *v = (int *)malloc(nv * sizeof(int));
    v[0] = 4;
    v[1] = 3;
    
    nv=4;
    v=(int*)remalloc(nv*sizeof(int));
    
    v[2] = 2;
    v[3] = 1;

    int sum = 0;
    for (size_t i = 0; i < nv; i++) {
        sum += v[i];
    }

    printf("%d\n", sum);

    free(v);
    return 0;
}
```

可以看到，使用malloc申请的内存空间，是需要free手动释放，这是危险的，容易造成内存泄漏和野指针，考虑对这一过程进行封装。



```cpp
//对于
size_t nv = 2;
int *v = (int *)malloc(nv * sizeof(int));
//封装成
std::vector<int>v(2);

//对于
nv=4;
v=(int*)remalloc(nv*sizeof(int));
//封装成
v.resize(4);
```



##### 实现

vector实现如下

```cpp
class myVector
{
private:
    int size;
    int *p;
public:
    int& operator[](int index)
    {
        if(index>=size||index<0)
        {
            std::cout<<"error,try to get the value out of boundary!!";
        }else
        {
            return *(p+index);
        }
    }
    
        friend std::ostream& operator<<(std::ostream& os,myVector& vec)
    {
        for(int i=0;i<vec.size;i++)
            os<<vec[i];
        return os;
    } 

    int resize(int size)
    {
        this->size=size;
        p=(int*)realloc(p,size);
    }

    myVector(int number):size(number)
    {
        p=(int *)malloc(size *sizeof(int));
        //构造函数的时候申请内存。
    }
    ~myVector()
    {
        free(p);//析构函数的时候释放内存
    }
};
```

然而这又出现了很多问题，就比如涉及拷贝赋值的时候

### 陷阱

```cpp
void show(myVector v)
{
    std::cout<<v<<'\n';
}

int main(char * str,char **args)
{
    myVector v(2);
    v[0]=4;
    v[1]=3;
    show(v);
    
    v.resize(4);
    v[2]=2;
    v[3]=1;
    show(v);


}
```

这样会报错

free(): double free detected in tcache 2
Aborted (core dumped)

为什么呢？

这是因为在调用show的时候，是浅拷贝，也就是说是只是原样拷贝了地址。这样当show结束的时候，里面的v被析构了，导致外面的v也整个被析构了。然后你还后面访问了v中的成员，就(core dumped)惹。
