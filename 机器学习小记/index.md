# 机器学习小记


# 监督学习
<script src="https://fastly.jsdelivr.net/npm/mathjax@2.7.5/MathJax.js?config=TeX-MML-AM_CHTML"></script>
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
  tex2jax: {
    inlineMath: [['$','$'], ['\\(','\\)']],
    displayMath: [['$$','$$'], ['\[','\]']],
    processEscapes: true,
    processEnvironments: true,
    skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
    TeX: { equationNumbers: { autoNumber: "AMS" },
         extensions: ["AMSmath.js", "AMSsymbols.js"] }
  }
});
</script>

<script type="text/x-mathjax-config">
  MathJax.Hub.Queue(function() {
    // Fix <code> tags after MathJax finishes running. This is a
    // hack to overcome a shortcoming of Markdown. Discussion at
    // https://github.com/mojombo/jekyll/issues/199
    var all = MathJax.Hub.getAllJax(), i;
    for(i = 0; i < all.length; i += 1) {
        all[i].SourceElement().parentNode.className += ' has-jax';
    }
});
</script>


监督学习从已有的数据集来建议一种模式（函数），来预测新的值，而数据是已有label的。

回归：从无限多的可能的数字来预测一个值  
[<img src="/images/assets/post8/huigui.PNG" width="100%"/>](/images/assets/post8/huigui.PNG)  

分类：只需对数据做类别预测，且是有限的  
[<img src="/images/assets/post8/fenlei.PNG" width="100%"/>](/images/assets/post8/fenlei.PNG) 

# 无监督学习
使用的数据集没用提前的属性，使用算法分析未标签化数据集，发现隐藏的模式或数据分组，无需人工干预，并形成聚类
聚类：  
[<img src="/images/assets/post8/julei.PNG" width="100%"/>](/images/assets/post8/julei.PNG) 

## 线性回归
使用一条线（函数）尽可能的拟合所有的数据  
[<img src="/images/assets/post8/xianxinghuigui.PNG" width="100%"/>](/images/assets/post8/xianxinghuigui.PNG) 

### 代价函数（Cost Function）
存在这样的函数关系：$$f_{w,b}(x^{(i)})=wx^{(i)}+b$$  
用$y^{(i)}$表示为真实值，则应使$f_{w,b}(x^{(i)})$(预测值)尽可能等于$y^{(i)}$    
通常使用右边这个式子来代表误差：$(f_{w,b}(x^{(i)})-y^{(i)})^{2}$。  
最终整个数据集得到的误差和的值会很大，所以尽可能的减小数字大小，在此基础上计算均方误差，并在除以2，就得到了代价函数:
$$J(w,b)=\frac{1}{2m} \sum\limits_{i = 0}^{m-1} (f_{w,b}(x^{(i)}) - y^{(i)})^2$$  

[<img src="/images/assets/post8/cost3d.PNG" width="100%"/>](/images/assets/post8/cost3d.PNG) 

### 梯度下降
通过代价函数可以得到，最终误差取决于w,b两个值，所以需要一点一点的修正w,b的值,这也是梯度下降算法：
$$ w=w-\alpha\frac{\partial J(w,b)}{\partial w}$$
$$ b=b-\alpha\frac{\partial J(w,b)}{\partial b}$$
这里$\alpha$叫做学习率，也决定了梯度下降的步幅，学习率过小则需要很多步才能收敛，过大则有可能起到反作用，$\frac{\partial J(w,b)}{\partial w}$为J(w,b)对w求偏导，也即斜率，斜率可正可负，正则最终w会越来越小，反而则越来越大,b同理。  
[<img src="/images/assets/post8/gra1.PNG" width="100%"/>](/images/assets/post8/gra1.PNG)  
[<img src="/images/assets/post8/gra2.PNG" width="100%"/>](/images/assets/post8/gra2.PNG)   
梯度下降算法涉及到的求导过程：
根据符合复合函数的求导法则,$f'(g(x))=f'(g(x))g'(x)$,先对w求偏导:
$$J'(w,b)=\frac{1}{2m} \sum\limits_{i = 0}^{m-1}\frac{\partial}{\partial w} ((f_{w,b}(x^{(i)}) - y^{(i)})^2) \tag{1}$$ 
$$J'(w,b)=\frac{1}{2m} \sum\limits_{i = 0}^{m-1}2 (f_{w,b}(x^{(i)}) - y^{(i)}) \frac{\partial}{\partial w} (wx^{(i)}+b - y^{(i)}) \tag{2}$$
$$J'(w,b)=\frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{w,b}(x^{(i)}) - y^{(i)}) \frac{\partial}{\partial w} (wx^{(i)}+b - y^{(i)}) \tag{3}$$
其中后半部分，对w求导，其余看作常数，对b求导，其余看作常数，得到：
$$\frac{\partial}{\partial w}(wx^{(i)}+b - y^{(i)}) =x^{(i)} \tag{4}$$
$$\frac{\partial}{\partial b}(wx^{(i)}+b - y^{(i)}) = 1 \tag{5}$$
带入（3）则得到：
$$\frac{\partial}{\partial w}J(w,b)=\frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{w,b}(x^{(i)}) - y^{(i)}) x^{(i)}$$
$$\frac{\partial}{\partial b}J(w,b)=\frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{w,b}(x^{(i)}) - y^{(i)}) $$
使用python实现上述算法：  
代价函数
```python

def compute_cost(x, y, w, b):
   
    m = x.shape[0] 
    cost = 0
    
    for i in range(m):
        f_wb = w * x[i] + b
        cost = cost + (f_wb - y[i])**2
    total_cost = 1 / (2 * m) * cost

    return total_cost

```
计算梯度：
```python
def compute_gradient(x, y, w, b): 
    m = x.shape[0]    
    dj_dw = 0
    dj_db = 0
    
    for i in range(m):  
        f_wb = w * x[i] + b 
        dj_dw_i = (f_wb - y[i]) * x[i] 
        dj_db_i = f_wb - y[i] 
        dj_db += dj_db_i
        dj_dw += dj_dw_i 
    dj_dw = dj_dw / m        
    dj_db = dj_db / m        
        
    return dj_dw, dj_db

```
其中： dj_dw=$\frac{\partial}{\partial w}J(w,b)$,dj_db=$\frac{\partial}{\partial b}J(w,b)$
### 多元线性回归
多元线性回归，样例具有多个特征$x^{(i)}$，结果预测受多个因素的影响：  
[<img src="/images/assets/post8/multihouse.PNG" width="100%"/>](/images/assets/post8/multihouse.PNG)  
可表示为：
$$ f_{w,b}(\mathbf{x}) =  w_0x_0 + w_1x_1 +... + w_{n-1}x_{n-1} + b $$
用向量点积表示为：
$$f_{w,b}(x)=\vec w \cdot \vec x+b$$
多元代价函数可表示为:
$$J(w,b)=\frac{1}{2m} \sum\limits_{i = 0}^{m-1} (f_{\vec w,b}(\vec x^{(i)}) - y^{(i)})^2$$ 
则梯度下降算法变为：  
$$ w_1=w_1-\alpha\frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{\vec w,b}(\vec x^{(i)}) - y^{(i)}) x_1^{(i)}$$
$$ \cdot\cdot\cdot $$
$$ \cdot\cdot\cdot $$
$$ \cdot\cdot\cdot $$
$$ w_n=w_n-\alpha\frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{\vec w,b}(\vec x^{(i)}) - y^{(i)}) x_n^{(i)}$$
由于b为常数，则：
$$ b=b-\alpha\frac{1}{m} \sum\limits_{i = 0}^{m-1} (f_{\vec w,b}(\vec x^{(i)}) - y^{(i)})$$
其中$f_{\vec w,b}(\vec x^{(i)})$为预测值，$y^{(i)}$为目标值。
### 特征缩放
归一化(normalization)与标准化(Standardization)是指特征缩放的过程，常见的方式：
min-max 归一化：$$x'=\frac{x-min(x)}{max(x)-min(x)}$$
Mean normalization: $$x'=\frac{x-\overline x}{max(x)-min(x)}$$
Standardization：$$x'=\frac{x-\overline x}{\sigma}$$
$\overline x$为平均数，$\sigma$为标准差  
问题：为什么需要进行特征缩放？  
$f(x) = w_1 * 1000 + w_2 * 0.5$,
其中$w_1$的变动对于整个结果的影响过大，$w_2$的影响被稀释,最终代价函数受$w_1$的影响更大，从而影响整个梯度下降。
### 线性回归实验代码
```python
# -*- coding:utf-8 -*-
import numpy as np
import matplotlib.pyplot as plt
from utils import *
import copy
import math

def myload_data():
    data = np.loadtxt("data/ex1data1.txt",delimiter=",")
    x = data[:,0]
    y = data[:,1]
    return x,y

def costfunction(x,y,w,b):
    total_cost = 0
    cost_sum = 0
    m = x.shape[0]
    for i in range(m):
        f_x=w*x[i]+b
        cost = (f_x-y[i])**2
        cost_sum += cost

    total_cost = (1 / (2 * m)) * cost_sum
    return total_cost
def gradientdescent(x,y,w,b):
    m = x.shape[0]
    dj_dw=0
    dj_db=0
    for i in range(m):
       tmpjb = w*x[i]+b-y[i]
       dj_db += tmpjb
       tmpjw = (w*x[i]+b-y[i])*x[i]
       dj_dw +=tmpjw
    
    dj_dw = (1/m)*dj_dw
    dj_db = (1/m)*dj_db
    return dj_dw,dj_db
def Learning(x,y,w_init,b_init,costfunction,gradientdescent,alpha,num_iters):

    m=len(x)
    temp_j=[]
    temp_w=[] 
    w = copy.deepcopy(w_init)
    b = b_init  
    iters_list=[]
    for i in range(num_iters):
        iters_list.append(i)
        dj_dw,dj_db = gradientdescent(x,y,w,b)
        w = w-alpha*dj_dw
        b = b-alpha*dj_db
        if i<100000:
            cost =costfunction(x,y,w,b)
            temp_j.append(cost)
            if(len(temp_j)>5):
                if(math.isclose(cost,temp_j[-1],abs_tol=0.0000001)&math.isclose(temp_j[-2],temp_j[-1],abs_tol=0.0000001)&math.isclose(temp_j[-3],temp_j[-2],abs_tol=0.0000001)):
                     print(f"Iteration {i-3:4}: Cost {float(temp_j[-4]):1.8f}   ")
                     print(f"Iteration {i-2:4}: Cost {float(temp_j[-3]):1.8f}   ")
                     print(f"Iteration {i-1:4}: Cost {float(temp_j[-2]):1.8f}   ")
                     print(f"Iteration {i}:Found Cost {cost:1.8f}   ")
                     break
                else:
                    if i% math.ceil(num_iters/10) == 0:
                        temp_w.append(w)
                        print(f"Iteration {i:4}: Cost {float(temp_j[-1]):1.8f}   ")

    plt.plot(iters_list,temp_j,c="r")
    plt.title("Cost vs. iterations")
    plt.ylabel('Cost J(w,b)')
    plt.xlabel('iterations')
    plt.show()
    plt.show()
    return  w, b, temp_j, temp_w

if __name__ == "__main__":
   
    x_train,y_train = myload_data()
    print ('The shape of x_train is:', x_train.shape)
    print ('The shape of y_train is: ', y_train.shape)
    print ('Number of training examples (m):', len(x_train))
    initial_w = 0
    initial_b = 0

    iterations = 9999
    alpha = 0.01
    
    w,b,_,_ = Learning(x_train,y_train,initial_w,initial_b,costfunction,gradientdescent,alpha,iterations)
    print("w,b found by gradient descent:", w, b)
    m = x_train.shape[0]
    predicted = np.zeros(m)

    for i in range(m):
        predicted[i] = w * x_train[i] + b

        predict1 = 3.5 * w + b
    print('For population = 35,000, we predict a profit of $%.2f' % (predict1*10000))

    predict2 = 7.0 * w + b
    print('For population = 70,000, we predict a profit of $%.2f' % (predict2*10000))    

    plt.plot(x_train, predicted, c = "b") 
    plt.scatter(x_train, y_train, marker='x', c='r') 
    plt.title("Profits vs. Population per city")
    plt.ylabel('Profit in $10,000')
    plt.xlabel('Population of City in 10,000s')
    plt.show()
```
## 逻辑回归

### 逻辑函数
对于分类问题，结果y的计算值为0或者为1，对于线性回归来讲，会产生过多的值，并且会小于0或者大于1，所以线性回归不适合这种情况。  
[<img src="/images/assets/post8/logistic.PNG" width="100%"/>](/images/assets/post8/logistic.PNG)  
g(z)也即y的结果始终在0~1区间，逻辑回归公式：
$$g(z)=\frac{1}{1+e^{(-z)}} $$
该函数可以看成是，在给定w,b前提下，对于输入x得到y等于1的概率大小，也可以这样表示：$f(x)=P(y=1|x;w,b)$

