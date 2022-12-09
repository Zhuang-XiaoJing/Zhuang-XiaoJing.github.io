---
title: 关于Matlab的parfor
date: 2022-09-26 17:16:06
tags: [matlab]
categories: [Notes]
cover: https://m1.im5i.com/2022/10/01/UFVTYG.jpg
top_img: 
---

## parfor 循环简介

MATLAB的parfor循环能够并行执行循环体中的一系列命令。

MATLAB客户端issue一个parfor命令给一群位于Parallel pool中的workers，让这群workers并行执行循环，从而提供比for循环更好的性能。需要注意的是：workers对于parfor的每个迭代都是独立计算的，即**不按照特定顺序执行迭代**，计算是**彼此独立**的（for循环的迭代是*sequential*的）。如果workers的数量等于循环迭代的次数，则每个worker执行一次循环迭代；如果循环次数多于worker数量，则一些workers执行不止一次的循环迭代；在这种情况下，为了减少通讯时间，一个worker一次可能接受多个迭代。

![img](https://ww2.mathworks.cn/help/parallel-computing/worker2.png)

### 使用parfor需要注意：

* 当循环中的迭代依赖于其他迭代结果时，不能使用parfor循环。每次迭代必须是独立的，且没有一定的计算顺序，这也是由于parfor循环的迭代分工给了不同的workers (no sharing information between iterations.)。因此parfor循环的迭代不能依赖于前面的迭代结果(唯一的例外是[Reduction Variables](https://ww2.mathworks.cn/help/parallel-computing/reduction-variable.html))。这意味着将for改为parfor时需要对代码做修改。

* 循环迭代必须是连续的，递增的整数值。

  以下声明会报错：

  ```matlab
  parfor i = 0:0.2:1      % not integers
  parfor j = 1:2:11       % not consecutive
  parfor k = 12:-1:1      % not increasing
  ```

  Fix errors hint:

  ```matlab
  iValues = 0:0.2:1;
  parfor idx = 1:numel(iValues) % numel returns number of array elements
  i = iValues(idx);
  ...
  end
  ```

* 不能在parfor-loop中嵌套另一个parfor-loop。

下面的代码给出同样的结果，这是因为每个元素只决定于循环变量，不依赖于其他变量。

```matlab
clear A
for i = 1:8
    A(i) = i;
end
A
A =

     1     2     3     4     5     6     7     8
```

```matlab
clear A
M = 8;
parfor (i = 1:8,M) %默认情况下,如果没有已打开的parallel pool,parfor自动开启parallel pool，且循环迭代是连续，递增的整数值,M specifies maximum number of workers
    A(i) = i; 
end
A
A =

     1     2     3     4     5     6     7     8
```




### 什么是Parallel Pool (并行池)?

并行池是[cluster](https://zh.wikipedia.org/wiki/%E8%AE%A1%E7%AE%97%E6%9C%BA%E9%9B%86%E7%BE%A4#:~:text=%E8%AE%A1%E7%AE%97%E6%9C%BA%E9%9B%86%E7%BE%A4%20%EF%BC%88%E8%8B%B1%E8%AA%9E%EF%BC%9A%20computer,cluster%20%EF%BC%89%E6%98%AF%E4%B8%80%E7%BB%84%E6%9D%BE%E6%95%A3%E6%88%96%E7%B4%A7%E5%AF%86%E8%BF%9E%E6%8E%A5%E5%9C%A8%E4%B8%80%E8%B5%B7%E5%B7%A5%E4%BD%9C%E7%9A%84%20%E8%AE%A1%E7%AE%97%E6%9C%BA%20%E3%80%82) (计算机集群，简单理解为核) 的一组 MATLAB workers：

![img](https://ww2.mathworks.cn/help/parallel-computing/pool1.gif)

### Parfor-loop中的variable classification问题

| Classification       | Description                                                  |
| -------------------- | ------------------------------------------------------------ |
| Loop Variables       | Loop indices                                                 |
| Sliced Variables     | Arrays whose segments are operated on by different iterations of the loop |
| Broadcast Variables  | Variables defined before the loop whose value is  required inside the loop, but never assigned inside the loop |
| Reduction Variables  | Variables that accumulates a value across iterations of the loop, regardless of the iteration order |
| Temporary  Variables | Variables created inside the loop, and not accessed outside the loop. |

上述所有的变量都体现在下面的代码中：

![img](https://ww2.mathworks.cn/help/parallel-computing/parfor_vartypes.png)

一个典型的问题是：在一个parfor-loop循环体内只能出现slice数组的一个元素，

```matlab
a(ii) = temp1;
a(ii+1) = temp2;
```

是不行的。另外slice变量的下标一定要连续，

```matlab
a(2*ii) = temp;
```

是不行的。

MATLAB使用临时变量不需要预定义，但使用parfor时，会弄混sliced变量和临时变量，导致出现 “...cannot be classified” 的问题。解决方法是在parfor中对临时变量进行预定义。一个典型的例子是：

**Invalid:**

```matlab
A = zeros(4, 10);
parfor i = 1:4
    for j = 1:10
        A(i, j) = i + j;
    end
    disp(A(i, 1)) % sliced数组A的元素A(i,1)已经在上面的循环体中出现了一回，无法区分A到底是sliced变量还是临时变量
end
```

**Valid:**

```matlab
A = zeros(4, 10);
parfor i = 1:4
    v = zeros(1, 10); % 预设临时变量v
    for j = 1:10
        v(j) = i + j;
    end
    disp(v(1))
    A(i, :) = v; % A的每个元素在parfor中只出现了一次，可判定为sliced变量
end
```

**Invalid**: 

下面是自己在计算phase diagram时遇到的例子：

```matlab
%% % --------------- winding # generator --------------
tic;
clear;clc;
a = 1;
w = -0.5;

tk = -1 + w;
tg = -1 - w;    % tk > tg
num = 50;
T3 = linspace(-6,6,num);
WW = linspace(-6,6,num);
windingMap = zeros(num);

parfor ii = 1:numel(T3)
    t3 = T3(ii);
    for jj = 1:numel(WW)
        ww = WW(jj);
        ty = t3 + ww;
        tp = t3 - ww;
        %         tk = -1-w;
        %         tg = -1+w;
        Wind = 0;
        delta_1 = 1e-9;     %求导步长
        delta_2 = 1e-4;     %积分步长
        kValues = -pi:delta_2:pi;
        for idx = 1:numel(kValues)
            k = kValues(idx);
            hk = tk+tg*exp(-1j*k*a)+ty*exp(-1j*k*2*a)+tp*exp(1j*k*a);
            log0 = log(hk);
            hk_delta = tk+tg*exp(-1j*(k+delta_1)*a)+ty*exp(-1j*(k+delta_1)*2*a)+tp*exp(1j*(k+delta_1)*a);
            log1 = log(hk_delta);
            Wind = Wind+(log1-log0)/delta_1*delta_2;
        end
        %         winding_number = round(abs(real(Wind/(2*pi*1j))));
        %         winding_map(jj,ii) = mod(winding_number,2);
        winding_number = Wind/(2*pi*1j);
        windingMap(ii,jj) = winding_number;
    end
end
```

上述程序报错为：“The variable windingMap in a parfor cannot be classiffied”。

**Valid 1** : 

修改为临时变量（先算一列列），使得windingMap的slice元素只在parfor中出现一次：

```matlab
parfor ii = 1:numel(T3)
    t3 = T3(ii);
    v = zeros(1,num)
    for jj = 1:numel(WW)
        ww = WW(jj);
        ty = t3 + ww;
        tp = t3 - ww;
        %         tk = -1-w;
        %         tg = -1+w;
        Wind = 0;
        delta_1 = 1e-9;     %求导步长
        delta_2 = 1e-4;     %积分步长
        kValues = -pi:delta_2:pi;
        for idx = 1:numel(kValues)
            k = kValues(idx);
            hk = tk+tg*exp(-1j*k*a)+ty*exp(-1j*k*2*a)+tp*exp(1j*k*a);
            log0 = log(hk);
            hk_delta = tk+tg*exp(-1j*(k+delta_1)*a)+ty*exp(-1j*(k+delta_1)*2*a)+tp*exp(1j*(k+delta_1)*a);
            log1 = log(hk_delta);
            Wind = Wind+(log1-log0)/delta_1*delta_2;
        end
        %         winding_number = round(abs(real(Wind/(2*pi*1j))));
        %         winding_map(jj,ii) = mod(winding_number,2);
        winding_number = Wind/(2*pi*1j);
        v(jj) = winding_number;
    end
    windingMap(ii,:) = v;
end

```

**Valid 2** :

将parfor嵌套在内层：

```matlab
for ii = 1:numel(T3)
    t3 = T3(ii);
    parfor jj = 1:numel(WW)
        ww = WW(jj);
        ty = t3 + ww;
        tp = t3 - ww;
        %         tk = -1-w;
        %         tg = -1+w;
        Wind = 0;
        delta_1 = 1e-9;     %求导步长
        delta_2 = 1e-4;     %积分步长
        kValues = -pi:delta_2:pi;
        for idx = 1:numel(kValues)
            k = kValues(idx);
            hk = tk+tg*exp(-1j*k*a)+ty*exp(-1j*k*2*a)+tp*exp(1j*k*a);
            log0 = log(hk);
            hk_delta = tk+tg*exp(-1j*(k+delta_1)*a)+ty*exp(-1j*(k+delta_1)*2*a)+tp*exp(1j*(k+delta_1)*a);
            log1 = log(hk_delta);
            Wind = Wind+(log1-log0)/delta_1*delta_2;
        end
        %         winding_number = round(abs(real(Wind/(2*pi*1j))));
        %         winding_map(jj,ii) = mod(winding_number,2);
        winding_number = Wind/(2*pi*1j);
        windingMap(ii,jj) = winding_number;
    end
end
```

在上面的例子中，使用parfor减少了约70%的时间。

## Plot

```matlab
plot(1,1,'bo')
```

[![UFFuF0.md.jpg](https://m1.im5i.com/2022/10/01/UFFuF0.md.jpg)](https://macimg.com/image/UFFuF0)

```matlab
plot(1,1:10,'bo')
```

[![UFF3KB.md.jpg](https://m1.im5i.com/2022/10/01/UFF3KB.md.jpg)](https://macimg.com/image/UFF3KB)

```matlab
plot(1:10,1,'bo')
```

[![UFF4Pz.md.jpg](https://m1.im5i.com/2022/10/01/UFF4Pz.md.jpg)](https://macimg.com/image/UFF4Pz)

```matlab
plot(1:10,1:10,'bo')
```

[![UFFIhs.md.jpg](https://m1.im5i.com/2022/10/01/UFFIhs.md.jpg)](https://macimg.com/image/UFFIhs)

```matlab
plot(1,reshape(1:9,[3,3])','bo')
```

报错：Vectors must be the same length.

```matlab
plot(ones(3),reshape(1:9,[3,3])','bo')
```

[![UFFm4W.md.jpg](https://m1.im5i.com/2022/10/01/UFFm4W.md.jpg)](https://macimg.com/image/UFFm4W)

```matlab
plot(reshape(1:9,[3,3])',reshape(1:9,[3,3])','bo')
```

[![UFFP0x.md.jpg](https://m1.im5i.com/2022/10/01/UFFP0x.md.jpg)](https://macimg.com/image/UFFP0x)

```matlab
plot(reshape(1:9,[3,3]),reshape(1:9,[3,3])','bo')
```

[![UFVDzQ.md.jpg](https://m1.im5i.com/2022/10/01/UFVDzQ.md.jpg)](https://macimg.com/image/UFVDzQ)

```matlab
plot(1:3,reshape(1:9,[3,3])','bo')
```

[![UFVUtq.md.jpg](https://m1.im5i.com/2022/10/01/UFVUtq.md.jpg)](https://macimg.com/image/UFVUtq)

```matlab
plot((1:4)',reshape(1:16,[4,4])','bo')
```



[![UFViYD.md.jpg](https://m1.im5i.com/2022/10/01/UFViYD.md.jpg)](https://macimg.com/image/UFViYD)

这样我们能理解[meshgrid](https://ww2.mathworks.cn/help/matlab/ref/meshgrid.html?searchHighlight=meshgrid&s_tid=srchtitle_meshgrid_1)函数：

```matlab
>> x = 0:3

x =

     0     1     2     3
     
>> y = 0:3

y =

     0     1     2     3
     
>> [X,Y]=meshgrid(x,y)

X =

     0     1     2     3
     0     1     2     3
     0     1     2     3
     0     1     2     3


Y =

     0     0     0     0
     1     1     1     1
     2     2     2     2
     3     3     3     3
     
>> plot(X,Y,'bo')
```

[![UFVxFy.md.jpg](https://m1.im5i.com/2022/10/01/UFVxFy.md.jpg)](https://macimg.com/image/UFVxFy)

meshgrid(x,y) 基于向量 x 和 y 返回二维网格坐标。X 是一个矩阵，**每一行是 x 的一个副本**；Y 也是一个矩阵，**每一列是 y 的一个副本**。即变量X存储2D格点坐标（grid-coordinates）中每个点的横坐标，Y存储图中每个点的纵坐标，相互的交叠就相当于"打网格"。

典型例子是可视化双变量函数，如 $f(x,y) = xe^{-x^2-y^2}$ 

```matlab
x = -2:0.25:2;y=x;
[X,Y] = meshgrid(x,y);
F = X.*exp(-X.^2-Y.^2);
surf(X,Y,F)
```

[![UFVE2p.md.jpg](https://m1.im5i.com/2022/10/01/UFVE2p.md.jpg)](https://macimg.com/image/UFVE2p)

