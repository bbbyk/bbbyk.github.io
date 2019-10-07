# delete 和 delete[]
对于简单类型，delete和delete []具有相同的效果，原因在于分配简单类型内存时，内存大小已经确定，系统可以记忆并且进行管理，在析构时，系统并不会调用析构函数。  
对于自定义类型数组，
```
class A
   {
   private:
      char *m_cBuffer;
      int m_nLen;
   public:
      A(){ m_cBuffer = new char[m_nLen]; }
      ~A() { delete [] m_cBuffer; }
   };
A *a = new A[10]
delete a;
delete []a;
```
使用delete a会释放a所指向的所有数组空间，但是析构只会调用A[0]的。也就是说对于A[1]~A[9]没有析构，其类中的m_cBuffer没有被删掉造成了内存泄露。  
而delete []a会让a中的每个成员都进行析构。  

**所以总结 都用delete [] 来删除数组不会出错**