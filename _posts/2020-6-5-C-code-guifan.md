---
layout: post
title:  "C代码自我编程规范"
date:   2021-12-13T14:25:52-05:00
header-img: img/post-bg-cook.jpg
author: Dandan
categories: C程序技术
tags:
    -  C程序技术
---
# 前言 
发现C程序没有一个标准的规范，就像if else后的`{`如果跟在if当前行或者单独起行都可，但是如果一个文件中有的`{`在单独行，有的在上边那一行，这样就看着很乱，或者一个人写的代码有的在同一行有的在下边，还是不一致，虽然无大碍，但还是别扭， 没办法， 哈哈， 有点代码洁癖，看不得不一致的东西。所以给自我做一个约束。

# 命名
## 变量的命名
对所有的宏、全局变量都要有详细的注释   ：功能、取值范围、以及其他的注意事项，注释切记与上边的代码行隔开
i、j、k对于局部变量是允许的
- 不建议大小写混写
- 建议驼峰命名法：驼峰命名法就是当变量名或函数名是由一个或多个单字连结在一起,而构成的唯一识别字时,第一个单字以小写字母开始,第二个单字的首字母大写;或每一个单字的首字母都采用大写字母,例如:myFirstName、myLastName,这样的变量名看上去就像骆驼峰一样此起彼伏,故而得名。
- 当标识符是一个单词时，首字母小写
- 不要使用拼音
- 全局变量应该前加g_  ,静态变量前应该加 s_
- 指针的 * 应该紧挨着变量
- 枚举应该用大写
- 常量名用大写
  
  
## 函数的命名
- 函数名应该以动词开头  因为函数是一组具有特 定功能的语句块。比如一个函数，它用于取得外部输入的数值，则可以命名为get_input_number。
- 
## 宏
全大写

# 代码间行数的间隔
## 独立的代码块 和变量之间必须加空行
例如：如下不符合规范
```c
if (!valid_ni(ni)) 
{ 
 ... // program code 
} 
repssn_ind = ssn_data[index].repssn_index; 
repssn_ni = ssn_data[index].ni;
```
应该书写如下：
```c
if (!valid_ni(ni)) 
{ 
 ... // program code 
} 

repssn_ind = ssn_data[index].repssn_index; 
repssn_ni = ssn_data[index].ni;
```


# 一行分多行写的规范
## 一长行语句划分多行
长表达式要在低优先级操作符处划分新行，
操作符放在新行之首，划分出的新行要进行适当的缩进，使排版整齐，语句可读。  
示例：
```c
示例：
perm_count_msg.head.len = NO7_TO_STAT_PERM_COUNT_LEN 
    + STAT_SIZE_PER_FRAM * sizeof( _UL ); 
 
act_task_table[frame_id * STAT_TASK_CHECK_NUMBER + index].occupied 
    = stat_poi[index].occupied; 

act_task_table[taskno].duration_true_or_false 
    = SYS_get_sccp_statistic_state( stat_item ); 

report_or_not_flag = ((taskno < MAX_ACT_TASK_NUMBER)
    && (n7stat_stat_item_valid (stat_item)) 
    && (act_task_table[taskno].result_data != 0));
```

## 循环、判断的长行
长表达式要在低优先级操作符处划分新行，操作符放在新行之首  
示例：
```c
if ((taskno < max_act_task_number) 
 && (n7stat_stat_item_valid (stat_item))) 
{ 
 ... // program code 
} 

for (i = 0, j = 0; (i < BufferKeyword[word_index].word_length) 
 && (j < NewKeyword.word_length); i++, j++) 
{ 
 ... // program code 

} 

for (i = 0, j = 0; 
 (i < first_word_length) && (j < second_word_length); 
 i++, j++) 
{ 
 ... // program code 
}
```

## 函数名或过程的参数过程，也应该划分
```c
n7stat_str_compare((BYTE *) & stat_object, 
    (BYTE *) & (act_task_table[taskno].stat_object), 
    sizeof (_STAT_OBJECT)); 
 
n7stat_flash_act_duration( stat_item, frame_id *STAT_TASK_CHECK_NUMBER 
    + index, stat_object );
```

# {}应用
if、for、do、while等语句独占一行，且执行语句部分无论多少都要加括号{}。 
示例：如下例子不符合规范。
```c
if (pUserCR == NULL) return; 
```
应如下书写：

```c
if (pUserCR == NULL) 
{ 
 return; 
}
```


# 注释
尽量写好注释
## 解决的bug单
解决问题单的代码处 标记处 对应的是解决哪个单字
```c
/*
*added by lidy at 2021/7/8
*解决从内核socket中读取不到部分br-lan、br-wan接口的问题，读取uci文件将这些接口重新添加一遍；
*/
```
或者 
```c
// 修复计算错误：更正了算法逻辑
// 作者：John Doe
// 日期：2023-06-16
```