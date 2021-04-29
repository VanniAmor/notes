挑战网址：http://www.pythontip.com/coding/code_oj_case/6

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

   