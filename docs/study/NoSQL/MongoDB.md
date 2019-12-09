# 【尚硅谷】前端视频_MongoDB夯实基础视频

> 废话不多说，先贴上视频链接
>
> https://www.bilibili.com/video/av21989676?p=8

## p6-尚硅谷_MongoDB入门-插入文档

1. db.\<collection\>.insert()

   如果插入多条,[{},{},{}]；

   插入单条{}，使用insertOne({})更清晰

   插入多条，使用insertMany([{},{},{}])更清晰。

   实际上都可以用insert()，只不过~One和~Many直观清晰

## p7-尚硅谷_MongoDB入门-查询文档

1. db.\<collection\>.find()

   默认查询所有；

   查询条件{}

## p8-尚硅谷_MongoDB入门-修改文档

1. db.\<collection\>.update({条件},{修改的值})

   + 默认直接用{修改的值}替换一整个原本的文档
   + 如果需要只对其中几个属性修改(没有该属性则添加)，需要**{$set:{修改的值}}**

   + $unset:{属性名:任意值}，删除该属性，所以值随意
   + **update默认只改符合条件的第一个**
   + 修改多个需要指定{muti:true}

2. db.\<collection\>.updateMany()

   + 默认修改多个

3. db.\<collection\>.updateOne()

   + 只修改一个

4. db.\<collection\>.replaceOne()
   + 替换一个

## p9-尚硅谷_MongoDB入门-删除文档

1. db.\<collection\>.remove({条件}) 和find一样
   + 默认和deleteMany(),删除所有符合条件的
   + 第二个参数\<justOne\>,只删除一个true/false, true即deleteOne
   + **如果remove({})直接全清空**，传递空对象，删除集合中所有文档，性能差，一条一条删的
   + 如果需要清空集合（直接**db.collection.drop()删除整个集合**，如果是最后一个集合，数据库也会没掉）
2. db.\<collection\>.deleteOne()
3. db.\<collection\>.deleteMany()
4. db.dropDatabase() 删除数据库

## p10-尚硅谷_MongoDB入门-练习1

1. 文档的属性如果还是文档，就把这个被包含的文档叫做内嵌文档

2. MongoDB**支持通过内嵌文档的属性作为条件进行查询**
   + "内嵌文档.内嵌文档的属性",**必须加上单引号or双引号**
   + 对内嵌文档的属性查询，对内嵌文档的内部字段匹配，**含有就行了**（比如模糊查询）
     + db.collection.find({'innerDoc.attr1':'wahaha'})查询collection集合中所有包含内嵌文档innerDoc及其属性attr1内含有‘wahaha’的文档。即这里只要attr1 == ‘wahaha’，或者attr1是个数组且含有'wahaha'也可以
   + $push和$addToSet,数组内添加
     + 如果数组中已经存在该元素，则$addToSet则不能添加($push可以重复)

## p11-尚硅谷_MongoDB入门-练习2

1. 