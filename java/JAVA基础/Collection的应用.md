## Java容器间相互继承关系

Java中有两种容器：Collection和Map

![img](https://www.runoob.com/wp-content/uploads/2014/01/java-coll-2020-11-16.png)



![在这里插入图片描述](https://img-blog.csdnimg.cn/20190301185638582.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2EyMDExNDgwMTY5,size_6,color_FFFFFF,t_70)

![JAVA各容器继承关系](https://img-blog.csdnimg.cn/20190213131312518.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjU3NDE0Mg==,size_16,color_FFFFFF,t_70#pic_center)

---

## HashMap的遍历



```
HashMap<String, Integer> map = new HashMap();

Set<String> keys = map.keySet();

// 1. 获取key的集合，增强for循环
for(String key : keys){
	sout（map.get(key)）	;
}


// 2. key的集合，使用迭代器
Iterator<String> iterator = keys.iterator();
while(iterator.hasNext()) {
	String key = iterator.next();
	sout(map.get(key));
}

// 3. 直接根据value循环，无法根据value获取key
Collection<Integer> Values = map.values();


// 4. Map.Entry循环, Map.Entry是HashMap类中的一个成员属性，类型是接口
Set<Map.Entry<String, Integer>> entries = map.entrySet();
Iterator<Map.Entry<String, Integer>> iterator = entries.iteratorr();
while(iterator.hasNext())
{
    Map.Entry<String, Integer> next = iterator.next();
    sout(next.getKey + "=" + next.getValue());
}

```







