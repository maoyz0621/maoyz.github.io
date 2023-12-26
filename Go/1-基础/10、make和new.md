# make 和 new

## 共同点

给变量分配内存



## 不同点

### 作用变量类型不同

new：string、int和数组分配内存

make：slice、map、channel分配内存

### 返回类型不同

new返回指向变量的指针

make返回变量本身

### 分配空间

new分配空间被清零

make分配空间后，会进行初始化

### 分配位置

make、new内存分配是在堆上还是在栈上？？？

