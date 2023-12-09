# 快速傅里叶变换（Fast Fourier Transform）



## 表示d阶多项式的方法

![image-20231203122351338](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203122351338.png)



### 值表示法的正确性

（d+1）个点定于了一个d阶的多项式

![image-20231203122821905](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203122821905.png)

将这d个式子写成矩阵的形式，如下：

由于x0, x1, ..., xd是唯一的，那么M是范德蒙德矩阵，可知M可逆，那么多项式唯一

![image-20231203123108418](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203123108418.png)



### 值表示法进行乘法比系数表示法快

![image-20231203122532020](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203122532020.png)



## FFT的思路

A和B都是d阶的，那么C = A * B 为 2d阶，只需要2d+1个不同的点，就可以还原出C来，因此只需要在A和B中找2d+1个不同的点，横坐标不变，纵坐标相乘，即可得到C的值表示法，再想办法转换成系数表示法。

一、如何把系数表示转换成值表示

二、如何把值表示转换成系数

![image-20231203122229096](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203122229096.png)



### 系数表示转换成值表示

最简单的方法就是随机选取n个点，代入多项式得到对应的值，但这种做法的时间复杂度是n^2

![image-20231203150932787](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203150932787.png)

考虑一个优化，若函数是对称的，它是奇函数或者偶函数，根据性质，只需要选一半的点，另一半自然就得到了

![image-20231203151455006](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203151455006.png)

那么对于一般的多项式，考虑如下做法：

- 将多项式分成 偶函数 Pe(x^2) + 奇函数 x * Po(x^2) 这种形式, 而且这样一分，每部分只有n/2-1阶，阶的维度少了一半
- 那么只需要计算 P(xi), P(-xi)自然也就知道了
- 对于这个例子， Pe(x^2) = 2x^4 + 7x^2 + 1又是一个多项式，那么递归即可完成

![image-20231203155946894](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203155946894.png)

但是这里有个问题，因为当我们计算完Pe(xi^2)之后，xi^2它没有相反数。

![image-20231203161419870](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203161419870.png)

考虑复数之后，如何选取复数？看下面这个例子

![image-20231203163237292](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203163237292.png)

可以看到，我们选取的复数是关于x^4 = 1的解，即1的四次方根

![image-20231203163354770](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203163354770.png)

**考虑一般多项式，我们想要将其转换为值表示，只需要找n个点，他们的横坐标为1的N次方根**

![image-20231203163614497](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203163614497.png)

1的n次方根可以被解释为复平面上沿着单位圆等距排布的一系列点，任一相邻两点的夹角是2pi / n

![image-20231203162757247](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203162757247.png)

**为啥n次方根可以用于递归过程？**

![image-20231203164447245](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203164447245.png)



### 系数表示法转值表示法伪代码

![image-20231203170812831](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203170812831.png)

橙色框框的推导

![image-20231203170616141](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203170616141.png)



### 值表示转换成系数表示（插值）

![image-20231203173530277](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203173530277.png)

![image-20231203173647908](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203173647908.png)

可以看到，若我们想要求系数，只需要求DFT矩阵的逆就好了

![image-20231203174058992](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203174058992.png)

![image-20231203174141250](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203174141250.png)

## 伪代码实现

![image-20231203174202838](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203174202838.png)



## C++ 递归版

![image-20231203212401394](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231203212401394.png)

```c++
#include<iostream>
#include<cstdio>
#include<cmath>
#include<complex>
using namespace std;
const int maxn = 3000005;
const double PI = acos(-1);
int n, m, k = 1;
complex<double>tmp[maxn], a[maxn], b[maxn];

void fft(complex<double>* f, int n, int type) {
	if (n == 1) return;
	for (int i = 0; i < n; i++) {
		tmp[i] = f[i];
	}
	// 奇偶分类
	for (int i = 0; i < n; i++) {
		if (i & 1) f[(n >> 1) + (i >> 1)] = tmp[i];	// Po
		else f[i >> 1] = tmp[i];	// Pe
	}
	complex<double>* g = f;	  //  Pe
	complex<double>* h = f + (n >> 1);  // Po
	fft(g, n >> 1, type);  // ye
	fft(h, n >> 1, type);  // yo
	complex<double> w(1, 0);
	complex<double> step(cos(2 * PI / n), type * sin(2 * PI / n));
	for (int i = 0; i < (n >> 1); i++, w *= step) {
		tmp[i] = g[i] + w * h[i];
		tmp[i + (n >> 1)] = g[i] - w * h[i];
	}
	for (int i = 0; i < n; i++) {
		f[i] = tmp[i];
	}
}

int main() {
	ios::sync_with_stdio(0); cin >> n >> m;
	for (int i = 0; i <= n; ++i)cin >> a[i];
	for (int i = 0; i <= m; ++i)cin >> b[i];
	while (k <= n + m)k <<= 1;  // 不足2的幂，高位补0
	fft(a, k, 1);
	fft(b, k, 1);
	for (int i = 0; i < k; ++i)a[i] *= b[i];
	fft(a, k, -1);
	for (int i = 0; i <= n + m; ++i)
		cout << int(a[i].real() / k + 0.5) << ' ';  // 除以n不要忘了
	return 0;
}
```



## C++迭代版

最终的序列以二进制来看，是原序列的二进制翻转，这叫rader排序

> rader排序中，r[i]可以由r[i >> 1]递推得出
>
> r[i] = r[i >> 1] >> 1
>
> if i mod 2 == 1
>
> ​	r[i]二进制中，最左边置为1

![image-20231205152118312](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231205152118312.png)

```c++
#include<iostream>
#include<cstdio>
#include<cmath>
#include<complex>
#include <algorithm>
using namespace std;
const int maxn = 3000005;
const double PI = acos(-1);
int n, m, k = 1;
complex<double>tmp[maxn], a[maxn], b[maxn];
int l;
int r[maxn];

void fft(complex<double>* f, int type) {
	for (int i = 0; i < k; i++) {
		if (i < r[i]) swap(f[i], f[r[i]]);
	}
	for (int mid = 1; mid < k; mid <<= 1) { // 待合并的区间中间
		complex<double> Wn(cos(PI / mid), type * sin(PI / mid)); //单位根 
		for (int j = 0, R = mid << 1; j < k; j += R) {  // R是区间右端点
			complex<double> w(1, 0);
			for (int k = 0; k < mid; k++, w = w * Wn) {  // 枚举左部分
				complex<double> x = f[j + k], y = w * f[j + mid + k];
				f[j + k] = x + y;
				f[j + k + mid] = x - y;
			}
		}
	}
}

int main() {
	ios::sync_with_stdio(0); cin >> n >> m;
	for (int i = 0; i <= n; ++i)cin >> a[i];
	for (int i = 0; i <= m; ++i)cin >> b[i];
	while (k <= n + m) {
		k <<= 1;  // 不足2的幂，高位补0
		l++;
	}
	for (int i = 0; i < k; i++) {
		r[i] = (r[i >> 1] >> 1) | ((i & 1) << (l - 1));
	}
	fft(a, 1);
	fft(b, 1);
	for (int i = 0; i < k; ++i)a[i] *= b[i];
	fft(a, -1);
	for (int i = 0; i <= n + m; ++i)
		cout << int(a[i].real() / k + 0.5) << ' ';  // 除以n不要忘了
	return 0;
}
```



# 快速数论变换（Fast Number-Theoretic Transform）

FFT 可以用来计算多项式乘法，但复数的运算会产生浮点误差。对于只有整数参与的多项式运算，有时，使用数论变换（Number-Theoretic Transform）会是更好的选择。

前置知识：乘法逆元、原根

## 乘法逆元

![image-20231205175128112](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231205175128112.png)

### 扩展欧几里得

适用于求单个元素的逆元，**a与p互质即可，p可以不是素数。**

> **贝祖定理**： 即如果a、b是整数，那么一定存在整数x、y使得ax+by=gcd(a,b)。
>
> 换句话说，如果ax+by=m有解，那么m一定是gcd(a,b)的若干倍。（可以来判断一个这样的式子有没有解）
>
> 有一个直接的应用就是 如果ax+by=1有解，那么gcd(a,b)=1；

```c++
// 朴素欧几里得
int gcd(int a,int b)
{
    return b == 0 ? a : gcd(b, a % b);

```

但是，对于上面的式子ax+by=m来说，我们并不仅仅想要知道有没有解，而是想要知道在有解的情况下这个解(x, y)是多少。

**扩展欧几里得定义：对于三个自然数 a,b,c ，求解 ax+by=c 的 (x,y)的整数解。**

![image-20231205174315937](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231205174315937.png)

```c++
// exgcd(a, b, x, y) 
// x = (x % p + p) % p
int exgcd(int a, int b, int& x, int& y) {
	if (b == 0) {
		x = 1; y = 0;
		return a;
	}
	int ans = exgcd(b, a % b, x, y);
	int temp = y;
	y = x - (a / b) * y;
	x = temp;
	return ans;
}
```



### 快速幂

![image-20231205183900043](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231205183900043.png)

```c++
// x = fast_power(a, p - 2, p)
int fast_power(int a, int b, int p) {
	int ans = 1;
	while (b) {
		if (b & 1) ans = ans * a % p;
		a = a * a % p;
		b >>= 1;
	}
	return ans;
}
```



### *线性递推

![image-20231205184901167](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231205184901167.png)

```c++
#include<iostream>
using namespace std;
const int maxn = 3e6 + 5;

int inv[maxn];

int main() {
	int n, p;
	cin >> n >> p;
	inv[1] = 1;
	for (int i = 2; i <= n; i++) {
		inv[i] = (p - p / i) * inv[p % i] % p;
	}
	for (int i = 1; i <= n; i++)
		cout << inv[i] << endl;
	return 0;
}
```



## 原根

### 阶

![image-20231206190827891](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206190827891.png)

![image-20231206191541264](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206191541264.png)

### 原根

原根是一种数学符号，设m是正整数，a是整数，若a模m的阶等于φ(m)，则称a为模m的一个原根。其中φ(m)表示m的欧拉函数

假设一个数g是P的原根，那么g^i mod P的结果两两不同，且有 1<g<P，0<i<P，归根到底就是g^(P-1) = 1 (mod P)当且仅当指数为P-1的时候成立.(这里**P是素数**)。

简单来说，g^i mod p ≠ g^j mod p （p为素数），其中i≠j且i, j介于1至(p-1)之间，则g为p的原根。

> 设*m*= 7，则φ（7）等于6。
>
> 设*a*= 2，由于2^3=8≡1(mod 7)，2^6=64≡1(mod7)，2^3≡2^6(mod7)，所以 2 不是模 7 的一个原根。
>
> 设*a*= 3，由于3^1≡3(mod 7)，3^2≡2(mod 7)，3^3≡6(mod 7)，3^4≡4(mod 7)，3^5≡5(mod 7)，3^6≡1(mod 7)，所以 3 是模 7 的一个原根。

### FFT需要用到单位复根的一些性质，原根也满足这些性质

![image-20231206200108465](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206200108465.png)

#### 性质一

![image-20231206200538163](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206200538163.png)

![image-20231206200041325](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206200041325.png)

![image-20231206200342371](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206200342371.png)

![image-20231206200314843](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206200314843.png)

#### 性质二

![image-20231206200629193](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206200629193.png)

#### 性质三

![image-20231206200702849](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206200702849.png)

#### 性质四

![image-20231206200905564](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231206200905564.png)



## NTT迭代版

p建议取998244353，它的原根为3。

![image-20231207155730901](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231207155730901.png)

看一下性质三，Wn取这个就好了

```c++
#include<iostream>
#include<cstdio>
#include<cmath>
#include <algorithm>
using namespace std;
const int maxn = 3000005;

typedef long long ll;
const ll p = 998244353, g = 3, gi = 332748118;  // g * gi % p = 1

int n, m, k = 1;
ll a[maxn], b[maxn];
int l;
int r[maxn];

ll qpow(ll a, ll b, ll p) {
	ll ans = 1;
	while (b) {
		if (b & 1) ans = ans * a % p;
		a = a * a % p;
		b >>= 1;
	}
	return ans;
}

void ntt(ll* f, int type) {
	for (int i = 0; i < k; i++) {
		if (i < r[i]) swap(f[i], f[r[i]]);
	}
	for (int mid = 1; mid < k; mid <<= 1) { // 待合并的区间中间
		ll wn = qpow(type == 1 ? g : gi, (p - 1) / (mid << 1), p);  // 单位根
		for (int j = 0, R = mid << 1; j < k; j += R) {  // R是区间右端点
			ll w = 1;
			for (int k = 0; k < mid; k++, w = w * wn % p) {  // 枚举左边
				ll x = f[j + k], y = w * f[j + k + mid] % p;
				f[j + k] = (x + y) % p;
				f[j + k + mid] = (x - y + p) % p;
			}
		}
	}
}

int main() {
	ios::sync_with_stdio(0); cin >> n >> m;
	for (int i = 0; i <= n; ++i)cin >> a[i];
	for (int i = 0; i <= m; ++i)cin >> b[i];
	while (k <= n + m) {
		k <<= 1;  // 不足2的幂，高位补0
		l++;
	}
	for (int i = 0; i < k; i++) {
		r[i] = (r[i >> 1] >> 1) | ((i & 1) << (l - 1));
	}
	ntt(a, 1);
	ntt(b, 1);
	for (int i = 0; i < k; ++i)a[i] = a[i] * b[i] % p;
	ntt(a, -1);
	ll inv = qpow(k, p - 2, p);
	for (int i = 0; i <= n + m; ++i) {
		cout << a[i] * inv % p << ' ';  // 除以n,变成乘上n的逆元
	}
	return 0;
}
```



## NTT递归版

```c++
#include<iostream>
#include<cstdio>
#include<cmath>
#include<complex>
using namespace std;

typedef long long ll;
const int maxn = 3000005;
const ll p = 998244353, g = 3, gi = 332748118;  // g * gi % p = 1

ll qpow(ll a, ll b, ll p) {
	ll ans = 1;
	while (b) {
		if (b & 1) ans = ans * a % p;
		a = a * a % p;
		b >>= 1;
	}
	return ans;
}

int n, m, k = 1;
ll tmp[maxn], a[maxn], b[maxn];

void ntt(ll* f, int n, int type) {
	if (n == 1) return;
	ll w = 1;
	ll wn = qpow(type == 1 ? g : gi, (p - 1) / n, p);
	// 奇偶分类
	for (int i = 0; i < n; i++) tmp[i] = f[i];
	for (int i = 0; i < n; i++) {
		if (i & 1) f[(i >> 1) + (n >> 1)] = tmp[i];
		else f[i >> 1] = tmp[i];
	}
	ll* g = f;				// Pe
	ll* h = f + (n >> 1);   // Po
	ntt(g, n >> 1, type);
	ntt(h, n >> 1, type);
	for (int j = 0; j < (n >> 1); j++, w = w * wn % p) {
		ll dummy = w * h[j] % p;
		tmp[j] = (g[j] + dummy) % p;
		tmp[j + (n >> 1)] = (g[j] + p - dummy) % p;
	}
	for (int i = 0; i < n; i++) f[i] = tmp[i];
}

int main() {
	ios::sync_with_stdio(0); cin >> n >> m;
	for (int i = 0; i <= n; ++i) cin >> a[i];
	for (int i = 0; i <= m; ++i) cin >> b[i];
	while (k <= n + m)k <<= 1;  // 不足2的幂，高位补0
	ntt(a, k, 1);
	ntt(b, k, 1);
	for (int i = 0; i < k; ++i) a[i] = a[i] * b[i] % p;
	ntt(a, k, -1);
	ll inv = qpow(k, p - 2, p);
	for (int i = 0; i <= n + m; ++i)
		cout << a[i] * inv % p << ' ';  // 除以n不要忘了
	return 0;
}
```



# 中国剩余定理CRT

**中国剩余定理(Chinese remainder theorem, CRT)**

![image-20231209143015043](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231209143015043.png)

考虑问题的分解：

**问题1：**计算一个整数 x ，使得它满足除以3余2、除以5余3、除以7余2。

如果能够找到三个整数 x1, x2, x3 ，使得：

- x1 除以3余2、除以5余0、除以7余0；
- x2 除以3余0、除以5余3、除以7余0；
- x3 除以3余0、除以5余0、除以7余2；

那么令 x = x1+x2+x3 ，就很容易验证这时的 x 就满足除以3余2、除以5余3、除以7余2。

分别称找到整数x1,x2,x3 的问题为**问题1-1**、**问题1-2**、**问题1-3**。可以看出这三个问题本质上是类似的。

下面对**问题1-1**继续分解，如果能够找到一个整数 y1 满足 y1 除以3余1、除以5余0、除以7余0，那么令 x1=2 × y1 ，就很容易验证这时的 x1 就满足除以3余2、除以5余0、除以7余0。

因此定义

**问题1-1-1**为：寻找整数 y1 满足 y1 除以3余1、除以5余0、除以7余0；

**问题1-2-1**为：寻找整数 y2 满足 y2 除以3余0、除以5余1、除以7余0；

**问题1-3-1**为：寻找整数 y3 满足 y3 除以3余0、除以5余0、除以7余1。

这三个问题本质上是相同的。

如果找到了 y1,y2,y3 ，那么就可以取 x = 2 × y1 + 3 × y2 + 2 × y3 。

下面就以**问题1-1-1**为例：寻找整数 z 使得 z 除以3余1、除以5余0、除以7余0。

于是 z 一定是 5×7=35 的倍数，假设 z=35k 。

那么就有 35k ≡ 1(mod3) ，而这时的 k 就是 5 × 7 模3的逆

![image-20231209143545154](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231209143545154.png)

![image-20231209143708968](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231209143708968.png)

```c++
#include<iostream>
using namespace std;

typedef long long ll;
const int maxn = 15;
int n, a[maxn], b[maxn];

void exgcd(ll a, ll b, ll& x, ll& y) {
	if (b == 0) {
		x = 1;
		y = 0;
	}
	else {
		exgcd(b, a % b, x, y);
		int tmp = y;
		y = x - (a / b) * y;
		x = tmp;
	}
}

int main() {
	ios::sync_with_stdio(0); 
	cin >> n;
	ll N = 1;
	for (int i = 0; i < n; i++) {
		cin >> a[i] >> b[i];
		N *= a[i];
	}
	ll ans = 0;
	for (int i = 0; i < n; i++) {
		ll x, y;
		exgcd(N / a[i], a[i], x, y);
		x = (x + a[i]) % a[i];
		ans = (ans + b[i] * (N / a[i]) * x) % N;
	}
	cout << ans;
	return 0;
}
```



# 任意模数NTT

 ![image-20231209162835363](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231209162835363.png)

![image-20231209162916948](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20231209162916948.png)

```c++
#include <algorithm>
#include <cstdio>
#include <cstring>
int mod;

namespace Math {
	inline int pw(int base, int p, const int mod) {
		static int res;
		for (res = 1; p; p >>= 1, base = static_cast<long long> (base) * base % mod)
			if (p & 1) res = static_cast<long long>(res) * base % mod;
		return res;
	}
	inline int inv(int x, const int mod) { return pw(x, mod - 2, mod); }
}

const int A = 998244353, B = 1004535809, C = 469762049;
const int G = 3, Ga = 332748118, Gb = 334845270, Gc = 156587350;
const long long AB = static_cast<long long> (A) * B;
const int inv_A = Math::inv(A, B), inv_AB = Math::inv(AB % C, C);

struct Int {
	int x1, x2, x3;
	explicit inline Int() { }
	explicit inline Int(int __x) : x1(__x), x2(__x), x3(__x) { }
	explicit inline Int(int __x1, int __x2, int __x3) : x1(__x1), x2(__x2), x3(__x3) { }
	static inline Int reduce(const Int& x) {
		return Int(x.x1 + (x.x1 >> 31 & A), x.x2 + (x.x2 >> 31 & B), x.x3 + (x.x3 >> 31 & C));
	}
	inline friend Int operator + (const Int& a, const Int& b) {
		return reduce(Int(a.x1 + b.x1 - A, a.x2 + b.x2 - B, a.x3 + b.x3 - C));
	}
	inline friend Int operator - (const Int& a, const Int& b) {
		return reduce(Int(a.x1 - b.x1, a.x2 - b.x2, a.x3 - b.x3));
	}
	inline friend Int operator * (const Int& a, const Int& b) {
		return Int(
			static_cast<long long> (a.x1) * b.x1 % A,
			static_cast<long long> (a.x2) * b.x2 % B,
			static_cast<long long> (a.x3) * b.x3 % C
		);
	}
	inline int get() {
		long long k1 = (static_cast<long long>(x2) - x1 + B) % B * inv_A % B;
		long long x4 = x1 + k1 * A;  // k1是mod B的, k1 * A是mod AB的, x4是mod AB的
		long long k4 = (x3 - x4 % C + C) % C * inv_AB % C; // k4是mod C的,等式右边每一项都需要在mod C下
		return (x4 + (k4 * (AB % mod)) % mod) % mod;
	}
};

#define maxn 131072

namespace Poly {
#define N (maxn << 1)
	int lim, s, rev[N];
	Int Wn[N | 1];
	void init(int n) {
		s = -1, lim = 1; while (lim <= n) lim <<= 1, ++s;
		for (register int i = 1; i < lim; ++i) rev[i] = rev[i >> 1] >> 1 | (i & 1) << s;
	}
	void NTT(Int* f, const int op = 1) {
		for (register int i = 1; i < lim; ++i) if (i < rev[i]) std::swap(f[i], f[rev[i]]);
		for (register int mid = 1; mid < lim; mid <<= 1) {
			const int t = lim / mid >> 1;
			int n = mid << 1;
			const Int wn(
				Math::pw(op ? G : Ga, (A - 1) / n, A),
				Math::pw(op ? G : Gb, (B - 1) / n, B),
				Math::pw(op ? G : Gc, (C - 1) / n, C)
			);
			for (register int j = 0, R = n; j < lim; j += n) {
				Int w = Int(1);
				for (int k = 0; k < mid; k++, w = w * wn) {
					Int x = f[j + k], y = w * f[j + k + mid];
					f[j + k] = x + y;
					f[j + k + mid] = x - y;
				}
			}
		}
		if (!op) {
			const Int ilim(Math::inv(lim, A), Math::inv(lim, B), Math::inv(lim, C));
			for (register Int* i = f; i != f + lim; ++i) *i = (*i) * ilim;
		}
	}
#undef N
}

int n, m;
Int a[maxn << 1], b[maxn << 1];
int main() {
	scanf("%d%d%d", &n, &m, &mod); ++n, ++m;
	for (int i = 0, x; i < n; ++i) scanf("%d", &x), a[i] = Int(x % mod);
	for (int i = 0, x; i < m; ++i) scanf("%d", &x), b[i] = Int(x % mod);
	Poly::init(n + m);
	Poly::NTT(a), Poly::NTT(b);
	for (int i = 0; i < Poly::lim; ++i) a[i] = a[i] * b[i];
	Poly::NTT(a, 0);
	for (int i = 0; i < n + m - 1; ++i) {
		printf("%d", a[i].get());
		putchar(i == n + m - 2 ? '\n' : ' ');
	}
	return 0;
}
```

