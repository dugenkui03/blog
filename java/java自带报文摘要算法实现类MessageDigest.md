##### 1. `MD5\SHA-1和SHA-256`

得到报文加密算法后的haxi数组分三步：
```java
String str="dugenkui";
String[] algo={"MD5","SHA-1","SHA-256"};

//1.返回实现了特定摘要算法的 报文摘要对象
MessageDigest md5 =MessageDigest.getInstance(algo[0]);

//2.使用特定的byte数组更新摘要
md5.update(str.getBytes());

//3.得到哈希值的字节数组
//通过执行最终操作（如填充）完成哈希计算。在调用完成后，摘要被重置)
byte[] res=md5.digest();
```

对于md5获取其他算法，我们希望结果值是16进制的字串。**<font color=red>byte(字节)由8位组成，16(hex)进制4位，因此每个byte需要两个16进制字符表示。</fotn>**

```java
    public static String byteArrayToHex(byte[] byteArray) {
        // 首先初始化一个字符数组，用来存放每个16进制字符
        char[] hexDigits = {'0','1','2','3','4','5','6','7','8','9', 'A','B','C','D','E','F' };

        // new一个字符数组，这个就是用来组成结果字符串的（解释一下：一个byte是八位二进制，也就是2位十六进制字符（2的8次方等于16的2次方））
        char[] resultCharArray =new char[byteArray.length * 2];

        // 遍历字节数组，通过位运算（位运算效率高），转换成字符放到字符数组中去
        int index = 0;
        for (byte b : byteArray) {
            resultCharArray[index++] = hexDigits[b>>> 4 & 0xf];
            resultCharArray[index++] = hexDigits[b& 0xf];
        }

        // 字符数组组合成字符串返回
        return new String(resultCharArray);
    }

```
