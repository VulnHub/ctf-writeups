### Solved by barrebas

We're given a strange algorithm that calculates the number of napkins some bloke needs when he eats `a` apples, `b` bananas, `c` cherries, and `d` donuts. Given 1 apple, 2 cherries and 3 donutes, the question is how many bananas does he need to eat to need 13,333,337 napkins?

Obviously, we're not going to solve this by looking at the algorithm. Instead, we modified the c++ source code to bruteforce the answer:

```c
#include<iostream>

using namespace std;

int test(int b){
	int a,c,d;
	int n = 0;
	//cin >> a >> b >> c >> d;
	a = 1; c = 2; d =3;
	if (d % 3 == 2) {
		for (int i=0;i<d/3;i++) {
			n += i/2;
		}
	}
	for (int i=0;a<b;a++) {
		n += 1;
	}
	while (c>1) {
		n += c;
		c -= 20;
	}
	n += a == b;
	while (c<5) {
		for (int i = 0;d<b;i++) {
			n += b;
			for (int e = 0;a==b;i++) {
				n--;
				b--;
			}
			b--;
			a--;
		}
		c++; //C++ :D
	}
	if (n >= 2689) {n -= 2689;}
	if (a+b+c+d > 50) {n=0;}
	//cout << n;
	return n;
}

int main()
{
	int i = 0;
	while(1)
	{
		int ret = test(i);
		if (ret == 13333337)
		{
			cout << i;
			return 0;
		}
		i++;
	}
	return 0;
}
```

This quicky yielded the answer:

```bash
bas@tritonal:~/backup/tmp$ g++ bf.cpp -o bf
bas@tritonal:~/backup/tmp$ ./bf
5164
```

That's a lot of bananas...

