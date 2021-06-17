挑战网址： http://www.pythontip.com/coding/code_oj_case/6



代码在线执行： https://tool.lu/coderunner

1. 求100以内的所有素数

   ```python
   print(' '.join("%s" % a for a in range(2, 100) if 0 not in( (a % b) for b in range(2, a) )) )
   ```

   - range，https://www.runoob.com/python3/python3-func-range.html
   - join， https://www.runoob.com/python/att-string-join.html
   - for i in range, https://blog.csdn.net/weixin_38705903/article/details/79238226
   - 列表推导式， https://www.cnblogs.com/zanao/p/11061738.html

2. 求a,b的最大公因数

   ```python
   a,b=max(a,b),min(a,b)
   while True : #求最大公约数用余数是否为0解决
       c = a%b
       if c==0: 
           print(b)
           break
       else:
           a = b
           b = c
   ```

3. 求a，b的最小公倍数

   ```python
   # 最小公倍数 = 两数之积 / 两数的最大公因数   
   a ,b = max(a,b),min(a,b)
   s = a%b
   t = a*b
    while s:
           a = b
           b = s 
           s = a%b
    
       answer = int(t/b)
   ```

4. 给你一个正整数列表 L, 判断列表内所有数字乘积的最后一个非零数字的奇偶性。如果为奇数输出1,偶数则输出0.。

   例如：L=[2,8,3,50]

   则输出：0

   ```python
   '''
   奇 * 奇 = 奇，偶 * 偶 = 偶， 奇 * 偶 = 偶
   解题思路：2与奇数相乘必是偶数，称为2将奇数中和
   但是2与5相乘为10，去掉0后为奇数，在本题中我称为2被5中和了
   '''
   # 题目变种， 给你一个正整数列表 L, 输出L内所有数字的乘积末尾0的个数。
   # 只需要判断所有数中因子2和5的总个数，最后乘积0的数字为总个数中小的一个
   
   c1,c2=0,0;
   for x in L:
    while x%2 == 0:
           x=x/2
           c1 +=1
    while x%5 == 0:
           x=x/5
           c2 +=1
   
   if(c1 > c2):
       print(0)
   else:
       print(1)
   ```

   

5. 银行在打印票据的时候，常常需要将阿拉伯数字表示的人民币金额转换为大写表示，现在请你来完成这样一个程序。

   在中文大写方式中，0到10以及100、1000、10000被依次表示为：    零 壹 贰 叁 肆 伍 陆 柒 捌 玖 拾 佰 仟 万

   以下的例子示范了阿拉伯数字到人民币大写的转换规则：

   1	壹圆

   11	壹拾壹圆

   111	壹佰壹拾壹圆

   101	壹佰零壹圆

   -1000	负壹仟圆

   1234567	壹佰贰拾叁万肆仟伍佰陆拾柒圆

   现在给你一个整数a(|a|<100000000), 请你打印出人民币大写表示.

   例如：a=1

   则输出：壹圆

   

   注意：请以**Unicode**的形式输出答案。提示：所有的中文字符，在代码中直接使用其Unicode的形式即可满足要求，中文的Unicode编码可以通过如下方式获得：u'壹'。

```python
trans_list1 = [u'零', u'壹', u'贰', u'叁', u'肆', u'伍', u'陆', u'柒', u'捌', u'玖']
trans_list2 = [u'圆', u'拾', u'佰', u'仟', u'万', u'拾', u'佰', u'仟', u'亿']
replace_list = [(u'零拾', u'零'), (u'零佰', u'零'), (u'零仟', u'零'), (u'零万', u'万'), (u'零零零',u'零'), (u'零零','零')]

(result, a) = (u'负', abs(a)) if a < 0 else ('', 0)

for index, str_n in enumerate(str(a)):
    result += trans_list1[int(str_n)] + trans_list2[len(str(a))-1-index]
for k,v in replace_list:
    result = result.replace(k, v)

print(u'零圆' if a < 0 else result.replace(u"零圆", u"圆"))
    
```

- replace: https://www.runoob.com/python/att-string-replace.html

- 三元表达式: https://blog.csdn.net/knidly/article/details/79964706



6. 我们经常遇到的问题是给你两个数，要你求最大公约数和最小公倍数。今天我们反其道而行之，给你两个数a和b，计算出它们分别是哪两个数的最大公约数和最小公倍数。输出这两个数，小的在前，大的在后，以空格隔开。若有多组解，输出它们之和最小的那组。注：所给数据都有解，不用考虑无解的情况。

   例如：a=3, b = 60

   则输出：12 15

```python
# 公式： 最小公倍数 / 最大公因数 = 两数之积
if a>b:
    a,b=b,a     #设定为a始终为小的那个数
list1=[]
list2=[]
list3=[]
for m in range(a,a*b+1):          #m从a一直试到a*b
    for n in range(m+1,a*b+1):       #n从m+1一直试到a*b
        if m%a==0 and n%a==0 and b%m==0 and b%n==0 and a*b==m*n:  #需同时满足所有这些条件
            list1.append(m)      #将符合条件的m值记录为列表1，n值记录为列表2，二者之和记录为列表3
            list2.append(n)
            list3.append(m+n)
 
j=-1
for i in list3:             #寻找列表3中最小值的索引，输出对应列表1和列表2中的m，n即为题目答案
     j+=1
     if i==min(list3):
         print(list1[j],list2[j])
```

7. 给你个小写英文字符串a和一个非负数b(0<=b<26), 将a中的每个小写字符替换成字母表中比它大b的字母。这里将字母表的z和a相连，如果超过了z就回到了a。

   例如a="cagy", b=3, 

   则输出 ：fdjb

```python
# 方法一
aim = ''
S = 'abcdefghijklmnopqrstuvwxyz'
for i in a:
    aim += S[(S.find(i)+b)%26]
print(aim)

# 方法二
ans = ''
for c in a:
    ans += chr((ord(c) - ord('a') + b) % 26 + ord('a'))
print(ans)
```

