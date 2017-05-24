---
title: 令我吃惊的代码片段
date: 2017-05-24 08:35
category: programming
tags:
---

##### [abs][1]
\\[
  abs(a) = \begin{cases}
    \sim a + 1 = \sim (a-1) & a \lt 0 \\\
    a & a \ge 0
  \end{cases}
\\]
```c
int32_t abs(int32_t x) {
    int32_t y = x >> 31;
    return (x+y) ^ y;
}
```

##### [add][2]
\\[ a + b \Leftrightarrow (a \verb| ^ | b) + (a \verb| & | b) \ll 1 \\]
```c
// eg: 1101 + 1011
//               1 1 0 1    (13)
//             + 1 0 1 1    (11)
// -----------------------
//               0 1 1 0
//           + 1 0 0 1
// -----------------------
//             1 0 1 0 0
//         + 0 0 0 1 0
// -----------------------
//           0 1 0 0 0 0
//       + 0 0 0 1 0 0
// -----------------------
//         0 0 1 1 0 0 0    (24)

int32_t add(int32_t x, int32_t y) {
    while (y) {
        int32_t t = x ^ y;
        y = (x & y) << 1;
        x = t;
    }
    return x;
}
```

##### [average][1]
```c
// (a + b) / 2 == ((a ^ b) + (a & b) << 1) / 2
//             == ((a ^ b) + (a & b) << 1) >> 1
//             == (a ^ b) >> 1 + (a & b)
int32_t average(int32_t x, int32_t y) {
    return (x&y) + ((x^y) >> 1);
}
```

##### [invSqrt][3]
```c
float invSqrt( float number )
{
	long i;
	float x2, y;
	const float threehalfs = 1.5F;

	x2 = number * 0.5F;
	y  = number;
	i  = * ( long * ) &y;                       // evil floating point bit level hacking
	i  = 0x5f3759df - ( i >> 1 );               // what the fuck?
	y  = * ( float * ) &i;
	y  = y * ( threehalfs - ( x2 * y * y ) );   // 1st iteration
//	y  = y * ( threehalfs - ( x2 * y * y ) );   // 2nd iteration, this can be removed

	return y;
}
```
##### [max][1]
```c
int32_t max(int32_t x, int32_t y) {
    int32_t m = (x-y) >> 31;
    return y & m | x & ~m;
}
```

##### [reverseBinaryTree][4]
```c
struct NormalNode {
  int value;
  struct NormalNode *left;
  struct NormalNode *right;
};

struct ReversedNode {
  int value;
  struct ReversedNode *right;
  struct ReversedNode *left;
};

struct ReversedNode *reverseBinaryTree(struct NormalNode *root) {
  return (struct ReversedNode *)root;
}
```


[1]: /assets/论程序底层优化的一些方法与技巧.pdf
[2]: https://news.ycombinator.com/item?id=9697008
[3]: https://leetcode.com/problems/sum-of-two-integers/
[4]: https://en.wikipedia.org/wiki/Fast_inverse_square_root
