- 数组介绍
- 基本数组
- 关联数组
- 案列分享



------



## 一、数组介绍

一个变量只能存一个值，但是现实中又又很多值需要存储，那么变量就有些拘谨了。比如做一个学员信息表，一个班50个人，每个人6条信息，我们需要定义300个变量才能完成。恐怖恐怖，这只是一个班的学生，一个学校呢？一个市呢？……我想静静了！

仔细想想上述的案例，一个学生六个信息:ID、姓名、性别、年龄、成绩、班级。可不可以定义六个变量就能存储这六类信息呢？答案是当然可以的！变量不行，我们就用数组。

## 二、基本数组

数组可以让用户一次赋予多个值，需要读取数据时只需通过索引调用就可以方便读出了。

### 2.1）数组语法

```
      数组名称=(元素1 元素2 元素3 ...)
```

### 2.2）数组读出

```
         ${数组名称}[索引]         索引默认是元素在数组中的排队编号，默认第一个从0开始
```

### 2.3）数组赋值

### 方法一: 一次赋一个值

array0[0]=’tom’

array0[1]=’jarry’

array0[2]=’natasha’

##### 方法二： 一次赋多个值

\# array2=(tom jack alice)

\# array3=(`cat /etc/passwd`) 希望是将该文件中的每一个行作为一个元素赋值给数组array3

\# array4=(`ls /var/ftp/Shell/for*`)

\# array5=(tom jack alice “bash shell”)

### 2.4）查看数组：

\# declare -a

declare -a array1=’([0]=”pear” [1]=”apple” [2]=”orange” [3]=”peach”)’

declare -a array2=’([0]=”tom” [1]=”jack” [2]=”alice”)’

### 2.5）访问数组元数：

\# echo ${array1[0]} 访问数组中的第一个元素

\# echo ${array1[@]} 访问数组中所有元素 等同于 echo ${array1[*]}

\# echo ${#array1[@]} 统计数组元素的个数

\# echo ${!array2[@]} 获取数组元素的索引

\# echo ${array1[@]:1} 从数组下标1开始

\# echo ${array1[@]:1:2} 从数组下标1开始，访问两个元素

### 2.6）遍历数组：

默认数组通过数组元素的个数进行遍历

```
[root@www ~]# echo ${array1[0]}pear[root@www ~]# echo ${array1[1]}apple[root@www ~]# echo ${array1[2]}orange[root@www ~]# echo ${array1[3]}peach
```

方法二： 针对关联数组可以通过数组元素的索引进行遍历

## 三、关联数组

关联数组可以允许用户自定义数组的索引，这样使用起来更加方便、高效。

### 3.1）定义关联数组

申明关联数组变量

\# declare -A ass_array1

\# declare -A ass_array2

### 3.2）关联数组赋值

方法一： 一次赋一个值

数组名[索引]=变量值

\# ass_array1[index1]=pear

\# ass_array1[index2]=apple

\# ass_array1[index3]=orange

\# ass_array1[index4]=peach

方法二： 一次赋多个值

\# ass_array2=([index1]=tom [index2]=jack [index3]=alice [index4]=’bash shell’)

### 3.3）查看数组：

\# declare -A

declare -A ass_array1=’([index4]=”peach” [index1]=”pear” [index2]=”apple” [index3]=”orange” )’

declare -A ass_array2=’([index4]=”bash shell” [index1]=”tom” [index2]=”jack” [index3]=”alice” )’

### 3.4）访问数组元素：

\# echo ${ass_array2[index2]} 访问数组中的第二个元数

\# echo ${ass_array2[@]} 访问数组中所有元数 等同于 echo ${array1[*]}

\# echo ${#ass_array2[@]} 获得数组元数的个数

\# echo ${!ass_array2[@]} 获得数组元数的索引

### 3.5）遍历数组：

通过数组元数的索引进行遍历,针对关联数组可以通过数组元素的索引进行遍历

```
[root@www ~]# echo ${ass_array2[index1]}tom[root@www ~]# echo ${ass_array2[index2]}jack[root@www ~]# echo ${ass_array2[index3]}alice[root@www ~]# echo ${ass_array2[index4]}bash shell
```

## 四、案列分享—-学员信息系统

```
/bin/bashfor ((i=0;i<3;i++))   do      read -p "输入第$((i + 1))个人名: " name[$i]      read -p "输入第$[$i + 1]个年龄: " age[$i]      read -p "输入第`expr $i + 1`个性别: " gender[$i]doneclear      echo -e "\t\t\t\t学员查询系统"while :   do      cp=0#      echo -e "\t\t\t\t学员查询系统"      read -p "输入要查询的姓名: " xm      [ $xm == "Q" ]&&exit      for ((i=0;i<3;i++))         do              if [ "$xm" == "${name[$i]}" ];then                  echo "${name[$i]} ${age[$i]} ${gender[$i]}"                  cp=1              fi      done      [ $cp -eq 0 ]&&echo "not found student"done
```



作业：

1. 使用关联数组统计文件/etc/passwd中用户使用的不同类型shell的数量
2. 使用关联数组按扩展名统计指定目录中文件的数量