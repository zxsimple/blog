> 最近在看吴军著的《数学之美》时对其中一篇《谈谈密码学的数学原理》颇感兴趣，平时对安全和加密相关的技术略有了解，知道诸如RSA一些加密算法通过公钥加密明文后可以通过私钥来解密密文。“公钥和密钥是两个完全不同的种子数据，公钥加密后的数据如何是被私钥解密的呢？”随着对文章的深入阅读，终于了解到了隐藏在其神秘面纱后面的真实原理。本文将围绕公钥加密算法，来探讨它基于费马小定理的实现。

### 公钥加密

公开密钥加密（public-key cryptography），也称为非对称(密钥)加密，该思想最早由雷夫·莫寇（Ralph C. Merkle）在1974年提出，之后在1976年。狄菲（Whitfield Diffie）与赫尔曼（Martin Hellman）两位学者以单向函数与单向暗门函数为基础，为发讯与收讯的两方创建密钥。 非对称密钥，是指一对加密密钥与解密密钥，这两个密钥是数学相关，用某用户密钥加密后所得的信息，只能用该用户的解密密钥才能解密。如果知道了其中一个，并不能计算出另外一个。因此如果公开了一对密钥中的一个，并不会危害到另外一个的秘密性质。称公开的密钥为公钥；不公开的密钥为私钥。 对称加密和解密过程是用一组同样的密钥，而非对称加密盒解密是用不同的一组密钥。

### 费马小定理

公钥加密有多种算法实现，例如RSA、ElGamal、背包算法、Rabin、Diffie-Hellman (D-H) 密钥交换协议中的公钥加密算法、Elliptic Curve Cryptography（ECC,椭圆曲线加密算法）。而其中使用最广泛的是RSA算法，RSA的公钥加密算法的数学基础是费马小定理。

**同余运算**

在介绍费马小定理之前，我先给大家介绍一个我们从来没有没有见过的数学运算符“≡”（除了数学专业和对数学涉猎广泛的同学们）。 两个整数a，b，若它们除以正整数m所得的余数相等，则称a，b对于模m同余 记作

```mathematica
a ≡ b mod (m)
```

读作a同余于b模m，或读作a与b关于模m同余。 比如26 ≡ 14 mod (12) 同余于的符号是同余相等符号 ≡。统一码值为 U+2261。

**费马小定理**

费马小定理有两种等价的表示 第一种是：

```mathematica
a^p ≡ a mod(p)
```

如果a是一个整数，p是一个质数，那么a^p – a 是p的倍数 如果a和p互为素数，那么第二种表示为：

```mathematica
a^(p-1) ≡ 1 mod(p)
```

**RAS加密算法**

1. 找两个很大的素数（质数）P 和 Q, 越大越好, 比如 100 位长的, 然后计算它们的乘积 N=P×Q, M=(P-1)×(Q-1)

2. 找一个和 M 互素的整数 E, 也就是说 M 和 E 除了 1 以外没有公约数

3. 找一个整数 D, 使得 E×D 除以 M 余 1, 即 E×D mod M = 1 加密过程是:

   ```mathematica
   Y = X^E mod N
   ```

4. 解密过程是:

   ```mathematica
   X = Y^D mod N
   ```

现在, 世界上先进的、最常用的密码系统就设计好了, 其中 E 是公钥谁都可以用来加密, D 是私钥用于解密, 一定要自己保存好. 乘积 N 是公开的, 即使别人知道了也没关系. 当然天下没有不能破解的密码，只是攻破一个密码所花费的代价有多少。而破解上面的加密算法就是通过N因式分解计算P和Q，而因式分解到目前的方法还是穷举获得因数P和Q。 90年代有人设计了一台价值将近100W美金的机器，这台机器破解64位的密钥需要34天，破解128位的密钥需要1018年。所以理论上来说只要破解一个加密算法需要耗费大量的时间，以至于此破解丧失了时效性，那么这个加密算法就是安全的。

### 加密算法实现

**大数运算**

我们在对一个两位数的因数P和Q做加密/解密运算时，会涉及到X^Y运算，而X和Y分别可能是一个四位数。比如

```
3204 ^ 3547 = 4912740121807952331801301397674357038549740553636606231811953453750285510587322…………………………………………………………………………………………………………………………………………..05588675043283796308759081385984
```

这个数足足有12435位长，Java中的基本数据类型甚至`BigInteger`都无法来表示并对其做运算的。因此对于次方运算和求余运算我们自己来实现，代码注释中有详细的计算原理和例子，相信仔细阅读后还是可以很快理解的。

#### 次方运算

```java
/**
 * Multiply number1 for n times, please make sure number1 = number2 in the beginning
 *
 * @param number1
 * @param number2
 * @param n
 * @return result
 */
public static String pow(String number1, String number2, long n) {

    if (n <= 1)
        return number1;
    n--;

    return pow(multiply(number1, number2), number2, n);

}


/**
 * Multiply of two big values
 *
 * @param number1
 * @param number2
 * @return
 */
public static String multiply(String number1, String number2) {

    boolean isMax = number1.length() >= number2.length();

    int len1 = number1.length() - 1;
    int len2 = number2.length() - 1;
    int arr1[] = new int[number1.length()];
    int arr2[] = new int[number2.length()];
    int arr[] = new int[isMax ? arr1.length * 2 : arr2.length * 2];

    int index = 0;

    while (true) {

        if (len1 < 0)
            break;

        arr1[index] = number1.charAt(len1) - '0';
        len1--;
        index++;

    }

    index = 0;

    while (true) {
        if (len2 < 0)
            break;
        arr2[index] = number2.charAt(len2) - '0';
        len2--;
        index++;
    }

    for (int i = 0; i < arr1.length; i++) {
        for (int j = 0; j < arr2.length; j++) {
            // example: 1234 * 123 最开始arr=[0,0,0,0,0,0,0,0];
            // 首先拿4 * 1 | 2 | 3 , 放到arr的0,1,2索引处 [12,8,4,0,0,0,0,0]
            // 然后拿3 * 1 | 2 | 3 , 放到arr的1,2,3索引处 [12,8+9,4+6,0+3,0,0,0,0]
            // ...
            // 最后得到 arr = [12,17,16,10,4,1,0,0];
            arr[j + i] += arr1[i] * arr2[j];
        }
    }

    for (int i = 0; i < arr.length; i++) {
        if (arr[i] > 9) {
            // arr = [12,17,16,10,4,1,0,0];
            // 首先得设置下一个数的值等于原值加上这个值进位的数值
            // 比如 [12,17,...],设置后为[2,17+(12/10=1)=18,...],
            // [1234,321,...] --> [1234%10=4, 321+(1234/10=123)=444,...]
            arr[i + 1] = arr[i + 1] + arr[i] / 10;
            arr[i] = arr[i] % 10;
        }
    }

    return convert(arr);
}
```

最后的convert方法将字符串反转并移初掉头部的0，很容易实现，我就不贴代码了。

#### 求余运算

```java
/**
 * Calculating the modulus of a large number 
 *
 * 同余定理
 * (a * b) % c = ((a % c) * (b % c)) % c
 * (a + b) % c = ((a % c) + (b % c)) % c
 *
 * example:
 * m=123
 * 123 = (1*10 + 2)*10 + 3
 * m%n = 123%n = (((1%n * 10%n + 2%n)%n * 10%n) % n + 3%n)%n
 *
 * @param input
 * @param modulo
 * @return
 */
public static long mod(String input, long modulo) {

    String reverse = reverse(input);
    long result = 0;
    long lastRowValue = 1;

    for (int i = 0; i < reverse.length(); i++) {

        if (i > 0) {
            lastRowValue = moduloBy(lastRowValue, modulo);
        }
        result += lastRowValue
                * Integer.parseInt((String) (reverse.substring(i, i + 1)));
    }

    result = result % modulo;
    return result;
}
```

### RSA加密解密

RSA类实现了加密，解密以及对公开数的破解。publicKey是公钥，privateKey是私钥，publicNumber为公开数。 在使用算法之前需要产生RSA的实例并调用init方法创始各参数。

```java
public class RSA {

    private long publicKey;
    private long privateKey;
    private long publicNumber;

    private final int DIGIT_NUMBER = 2;

    /**
     * encrypt the input value x
     *
     * @param x
     * @return encrypted value
     */
    public long encrypt(long x) {

        System.out.println(x + " ^ " + publicKey + " % " + publicNumber);
        return BigMath.mod(

                BigMath.pow(String.valueOf(x), String.valueOf(x), publicKey),
                publicNumber);

    }
    /**
     * decrypt the input value y
     *
     * @param y
     * @return decrypted value
     */
    public long decrypt(long y) {

        System.out.println(y + " ^ " + privateKey + " % " + publicNumber);
        return BigMath.mod(
                BigMath.pow(String.valueOf(y), String.valueOf(y), privateKey),
                publicNumber);
    }


    /**
     * initialize the algorithm
     */
    public void init() {

        long p = Maths.randPrime(DIGIT_NUMBER);
        long q = Maths.randPrime(DIGIT_NUMBER);

        if (p == q || p == 2) {

            init();
        } else {

            publicNumber = p * q;
            long m = (p - 1) * (q - 1);
            publicKey = Maths.randPrime(m, DIGIT_NUMBER);
            privateKey = calculatePrivateKey(m, publicKey);

        }
    }

    private long calculatePrivateKey(long m, long e) {


        for (long d = 1; d < Integer.MAX_VALUE; d++) {

            if (e * d % m == 1)
                return d;
        }

        return -1;
    }

    public void crack() {

        for (int i = 2; i <= Math.sqrt(publicNumber); i++) {

            long n = BigMath.mod(String.valueOf(publicNumber), i);
            long m = -1;

            if (n == 0) {
                m = publicNumber / i;
                System.out.println(i + "," + m);
            }
        }
    }
}
```

**算法执行**

假如我们选取两个两位数作为算子P=97, Q=73，那么公开数是7081，M=6912；根据RSA算法，选取11作为公钥，5027作为私钥。 对整数2加密：2 ^ 11 % 7081 = 2048，密文是 2048 对于密文解密：2048 ^ 5027 % 7081 = 2 ，得到解密后的值是 2

### 参考资料

[费马小定理](https://zh.wikipedia.org/wiki/%E8%B4%B9%E9%A9%AC%E5%B0%8F%E5%AE%9A%E7%90%86)

[同余](https://zh.wikipedia.org/wiki/%E5%90%8C%E9%A4%98)

[费马小定理在公钥加密中的应用及原理](https://www.cnblogs.com/technology/archive/2011/02/04/1949086.html)

[web2.0calc](https://web2.0calc.com/)

[Big Integer Calculator: 100 digits! A million digits?!](http://www.javascripter.net/math/calculators/100digitbigintcalculator.htm)

