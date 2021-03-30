## **magic_quotes_gpc**的使用



在php.ini中 设置 magic_quotes_gpc=on 将单引号转换为\'的转义字符

通过函数 get_magic_quotes_gpc 来获取上述配置



**本函数已自 PHP 7.4.0 起*废弃*。强烈建议不要使用本函数。在php7.4起，该函数一直返回false**



该配置项会对web传入参数进行转义，需要 stripslashes 进行反转义