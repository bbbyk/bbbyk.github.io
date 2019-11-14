# 使用using改变个别成员的可访问性
```
转载:https://www.cnblogs.com/nowhszh/p/11194039.html
```
**using改变成员的可访问性会带来歧义,尽量避免使用**  
### 1.通过using将用户对基类成员的访问权限复原  
```
class Base
{
public:
    int base_public;
protected:
    int base_protect;
private:
    int base_private;
};

class Derive :private Base
{
public:
    using Base::base_public;
protected:
    using Base::base_protect;
private:
    //using Base::base_private; // error , 编译器报错 , 不可访问
};
```
  
### 2.通过using声明 , 做出一些不合常规的代码
```
#pragma warning (disable:4996)
#include <iostream>

class Base
{
public:
    int base_public = 1;
protected:
    int base_protect = 2;
private:
    int base_private = 3;
};

class Derive :private Base
{
public:
    //在public作用域声明基类中的成员
    using Base::base_public;
    using Base::base_protect;
    //using Base::base_private; // error , 编译器报错 , 不可访问
};

int main()
{
    Derive test;
    std::cout << "基类的公有成员: "<< test.base_public << std::endl;
    std::cout << "基类的保护成员: " << test.base_protect << std::endl;
    //std::cout << "基类的私有成员: " << test.base_private << std::endl; //error , 不可访问

    system("pause");
    return EXIT_SUCCESS;
}
```
代码中通过给基类的成员设定了默认初始值以便访问时显示 , 最终的运行结果如下  

基类的公有成员: 1
基类的保护成员: 2
请按任意键继续. . .  
  
  
代码中派生类继承时同样使用private的方式继承 , 然而我们把using声明放在了public权限下  , 由于 " using声明语句中名字的访问权限由该using声明语句之前的访问说明符来决定" 

所以此时基类中的公有成员和保护成员都变成了派生类中的公有成员  , 也就是说通过using声明 , 我们通过私有继承的派生类 比 公有继承的访问权限反而更高  ! 

