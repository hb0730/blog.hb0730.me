---

title: Mysql大小写敏感

date: 2020-09-24 10:46:49

author: hb0730

authorLink: https://blog.hb0730.com

tags: ['Mysql']

categories: [ 'Mysql' ]

---

# 如何查看大小写敏感

`show variables like 'lower%'`

| Variable_name          | Value |
| ---------------------- | ----- |
| lower_case_file_system | OFF   |
| lower_case_table_names | 0     |

**lower_case_table_names**: 用于区分大小写

+ **0**：区分大小写，
+ **1**：不区分大小写

所需只需要**lower_case_table_names**修改成功**0**后重启服务即可
