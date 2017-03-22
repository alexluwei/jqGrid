# jqGrid
jqGrid项目使用总结

最近项目中用jqGrid来实现页面中的表格数据，使用过程中感触颇多，总体发现jqGrid灵活性还是很好的，我使用过程中参考了API文档，感觉这个API文档挺全面的，文档地址为：
[jqGrid实例中文版](http://blog.mn886.net/jqGrid/) 

首先说一下项目的需求：<br>
type==0  永久锁     status==0 正常    --可以变更为定期激活锁<br>
type==1  定期激活锁  status==0 正常 || status==1 未激活   --可以变更为永久锁，可挂失<br>
status==4  已挂失  --只有已挂失的锁可以取消挂失

下面是我在项目中遇到问题的总结：

#### 1.onSelectRow的问题

```javascript
onSelectRow: function(rowid, status){
    if(status){
        //选中行时执行的操作
    }else{
        //取消选中行时执行的操作
    }    
}
```
我的项目中的需求是选中一行或者多行时，要根据每一行的某两个值判断是否可以点击上面的按钮，如果可以，某个按钮会变亮可点，如果不可以，某个按钮会变暗不可点，具体效果如图所示

![](https://github.com/FloweringVivian/jqGrid/blob/master/images/readme.jpg) 

onSelectRow有两个参数，rowid代表行id，status代表选中状态，选中时为true，取消选中时为false，所以我需要在选中行时把行的数据push到一个数组里，然后取消选中行时在数组里根据行id找到这条数据，然后删除，然后写一个方法来循环这个数组中的数据，进行按钮的是否可点击判断，方法如下：

```javascript
//控制按钮状态
function watchBtn(arrList){
  var toForeverBtn = false;
  var toActivationBtn = false;
  var lossBtn = false;
  var cancelLossBtn = false;
  if(arrList && arrList.length>0){
    for(var i=0;i<arrList.length;i++){
      if(arrList[i].type != 1 || arrList[i].status != 0){  //只有type==1 && status==0(定期激活 正常)才能变更为永久
        toForeverBtn = true;
      };
      if(arrList[i].type != 0 || arrList[i].status != 0){  //只有type==0 && status==0(永久 正常)才能变更为定期激活
        toActivationBtn = true;
      };
      if(arrList[i].type != 1 || (arrList[i].status != 0 && arrList[i].status != 1)){  //只有type==1 && (status==0 || status==1)(定期激活 正常 未激活)才能挂失
        lossBtn = true;
      };
      if(arrList[i].status != 4){  //只有status==4(已挂失)才能取消挂失
        cancelLossBtn = true;
      };
    }
  }else{
    toForeverBtn = false;
    toActivationBtn = false;
    lossBtn = false;
    cancelLossBtn = false;
  }
  //四个按钮状态
  $("#toForever").prop("disabled",toForeverBtn);
  $("#toActivation").prop("disabled",toActivationBtn);
  $("#loss").prop("disabled",lossBtn);
  $("#cancelLoss").prop("disabled",cancelLossBtn);
}
```

#### 2.formatter的问题
我的项目中有个这样的需求，就是接口会返回类型type和状态status，返回的是数值，但是不会返回对应的名称，所以需要我自己根据返回的数值显示出对应的名称，于是我查了一下API，发现formatter可以实现，于是我就这样写：

```javascript
colModel:[
    {
        label:"激活类型",
        name: "type",
        width: 100,
        formatter: function(cellValue, options, rowObject){
            switch(cellValue){
                case 0:
                    return "永久";
                    break;
                case 1:
                    return "定期激活";
                    break;
           }
        }
    }
]
```

status也采用了同样的原理，结果问题来了，加了formatter以后，把原来的type和status都赋值成了文本内容，就是formatter里return的值，但是我之前的控制按钮状态的方法是根据type和status的数值来判断的，结果现在判断就不生效了，后来发现了一个解决办法，就是不直接改变type的值，而是增加一个参数typeName，让type列隐藏（hidden:true），然后给typeName列写formatter，所以说jqGrid还是很灵活的，代码如下，具体代码请查看我的项目源代码

```javascript
colModel:[
    {
        label:"激活类型",
        name: "type",
        width: 100,
        hidden: true
    },
    {
        label:"激活类型",
        name: "typeName",
        width: 100,
        formatter: function(cellValue, options, rowObject){
            switch(rowObject.type){
                case 0:
                    return "永久";
                    break;
                case 1:
                    return "定期激活";
                    break;
           }
        }
    }
]
```

formatter有三个参数cellValue(当前cell的值)，options(该cell的options设置，包括{rowId, colModel,pos,gid})，rowObject(当前cell所在row的值，如{ id=1, name="name1", price=123.1, ...})，所以我可以根据rowObject.type来确定typeName的值