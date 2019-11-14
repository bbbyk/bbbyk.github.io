# 类中函数定义后接const
```
class A{
private: int a;
public:
    A(): a(0) {}
    int get() const {return a;}
    int Get() {return a;}
};
```
const在此中的作用是将隐藏传入函数的this标记为const，不让在函数内部更改this指向的本实体的属性。  
在调用的时候，const函数可以被const和const对象调用，而不加const的函数只能被非const对象调用。  
```
const A a1;
A a2;
a1.get();
a2.get();
a1.Get(); //出错 cosnt函数只能够被非const对象调用
a2.Get();
```
