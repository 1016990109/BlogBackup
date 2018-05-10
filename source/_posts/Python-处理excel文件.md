---
title: Python 处理excel文件
date: 2018-05-09 22:42:58
tags:
    - python
---

最近需要用到Python去处理excel，目标是根据用户输入的信息，先显示每列的列名以及这一列示例行（取前两行信息），然后根据输入的列号删除对应的列。

网上搜索了下主要有几种方案：

### 1. 使用xlrd、xlwt、xlutils组合

这种方案比较常见，而且读取和写入速度较快，但是只能操作2003版本之前的xls文件，处理不了xlsx，所以想要处理2003版本之后的请绕道。

> 注意！  
xlutils复制的excel格式上会存在一些问题，我在使用的时候就因为这个原因而弃用了，灰底的表格会变成深蓝色底的，难以接受。 

<!-- more -->

xlrd用来读取excel内容，xlwt写入excel内容，xlutils封装了一些常用的操作excel的函数供用户使用，一些使用如下：

```python
#打开excel文件
workbook = xlrd.open_workbook('myexcel.xls')
#获取表单
worksheet = workbook.sheet_by_index(0)
#读取数据
data = worksheet.cell_value(0,0)
##另一种获取数据，但这种是包含数据类型的，需要内容可通过value获取
data2 = worksheet.cell(0, 0)
#----xlwt库
#新建excel
wb = xlwt.Workbook()
#添加工作薄
sh = wb.add_sheet('Sheet1')
#写入数据
sh.write(0,0,'data')
#保存文件
wb.save('myexcel.xls')
#----xlutils库
#打开excel文件
book = xlrd.open_workbook('myexcel.xls')
#复制一份
new_book = xlutils.copy(book)
#拿到工作薄
worksheet = new_book.getsheet(0)
#写入数据
worksheet.write(0,0,'new data')
#保存
new_book.save()
```

### 2. 使用openpyxl

与xlrd、xlwt、xlutils组合不同，openpyxl只支持excel 2003之后的也就是xlsx文件，示例如下：

```python
import openpyxl
# 新建文件
workbook = openpyxl.Workbook() 
# 写入文件
sheet = workbook.activesheet['A1']='data'
# 保存文件 
workbook.save('test.xlsx')

#或者
#打开文件
wb = openpyxl.load_workbook('test.xlsx')
#读取数据
ws = wb.active
cols = ws.columns
cols[0][1].value = 'data'
```

### 3. 使用win32com操作系统的Excel程序

这种方式约束较多，要求Windows + Microsoft Excel，是使用win32com库来操作Excel程序来完成对excel文件的操作，官方的API特别的复杂，要找到自己需要的API还是要费一番功夫的，但是相应的能做的事也就多了，你能在Excel程序中做的事，几乎使用win32com都能做，具体可用的可查看官方的[文档](https://documentation.devexpress.com/OfficeFileAPI/12078/Spreadsheet-Document-API/Examples/Worksheets)，几乎和vb程序差不多，不用看c++的。
举一些我在项目中用到的例子：

```python
#打开应用
xlApp = win32com.client.Dispatch('Excel.Application')
#打开表格
xlBook = xlApp.Workbook.Open('test.xls')
#保存文件
xlBook.Save()
#另存为
xlBook.SaveAs('new.xls')
#关闭文件
xlBook.Close(SaveChanges=0)
#获得sheet，传入sheet名字
sht = xlBook.Worksheets('Sheet1')
#获得sheet2，传入index
sht = xlBook.worksheets[index]
#获得单元格内容
data =  sht.Cells(row, col).Value
#设置单元格内容
sht.Cells(row, col).Value = 'New Value'
#获得一块
myRange = sht.Range(sht.Cells(row1, col1), sht.Cells(row2, col2)).Value
#选择sheet
sht.Activate
#获得一列
sht.Columns(index)
#删除一列
sht.Columns(1).Delete()
```

## 总结

以上就是3种常用操作excel的python库，读者可根据自己的情况选择合适的库。
