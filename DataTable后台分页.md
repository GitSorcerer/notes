# 后台分页

## POM

```properties
 <!--PageHelper-->
    <dependency>
      <groupId>com.github.pagehelper</groupId>
      <artifactId>pagehelper</artifactId>
      <version>5.0.3</version>
    </dependency>

    <!--PageHelper依赖-->
    <dependency>
      <groupId>com.github.jsqlparser</groupId>
      <artifactId>jsqlparser</artifactId>
      <version>1.0</version>
    </dependency>
```

## sqlMapConfig.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <setting name="cacheEnabled" value="true" /><!-- 全局映射器启用缓存 -->
        <setting name="useGeneratedKeys" value="false" /><!-- 设置是否使用JDBC的getGenereatedKeys方法获取主键并赋值到keyProperty设置的领域模型属性，默认false -->
        <setting name="defaultExecutorType" value="REUSE" /><!-- 配置默认的执行器。SIMPLE执行器没有什么特别之处，REUSE执行器重用预处理语句，BATCH和批量更新 -->
        <!--<setting name="mapUnderscoreToCamelCase" value="true" />&lt;!&ndash; 驼峰转换 &ndash;&gt;-->
        <setting name="jdbcTypeForNull" value="NULL" /><!-- mybatis在oracle插入空值时报错问题  http://blog.csdn.net/fishernemo/article/details/27649233 -->
        <setting name="logImpl" value="LOG4J" /><!-- 配置日志输出方式为log4j -->
    </settings>

    <typeAliases>
        <package name="com.zccs.assets_management.pojo"/>
    </typeAliases>

    <!-- PageHelper分页插件 -->
    <plugins>
        <plugin interceptor="com.github.pagehelper.PageInterceptor"/>
    </plugins>

</configuration>
```

## 后台代码

```java
 /**
     * 分页
     *
     * @param page  当前页
     * @param limit 每页显示的数量
     * @param start 开始的记录序号
     * @return
     */
    @ControllerLog(memo = "分页查询")
    @RequestMapping(value = "/empList")
    public Map<String, Object> empList(@RequestParam(value = "page", defaultValue = "1", required = false)
                                               Integer page,
                                       @RequestParam(value = "limit", defaultValue = "10", required = false)
                                               Integer limit,
                                       @RequestParam(value = "start", defaultValue = "10", required = false)
                                               Integer start,
                                       HttpServletRequest request) {
        Map<String, Object> resultMap = new HashMap<>(6);
        //查询条件
        String searchCondition = request.getParameter("searchCondition");
        //显示的所有列  和 排序一起使用
        String col = request.getParameter("col");
        //排序  列的序号 以及排序规则
        String order = request.getParameter("order");
        //搜索框
        String search = request.getParameter("search[value]");
        String[] columns = new String[0];
        if (!"".equals(col)) {
            columns = col.split("~");
        }
        BaseUtils.loggerDebug(search);
//        BaseUtils.loggerDebug("" + JSON.toJSONString(map));
        BaseUtils.loggerDebug("page:" + page + "  limit:" + limit + "  start:" + start);
        //page:1  limit:10  start:10

        try {
//            Page<EmpModel> pageObj = PageHelper.offsetPage(page * limit, limit, true);
            //当前页从0开始
            Page<EmpModel> pageObj = find(page + 1, limit, search, searchCondition, columns, order);
            if (pageObj == null) {
                resultMap.put("msg", "暂无数据!");
            } else {
                int pages = pageObj.getPages();
                long total = pageObj.getTotal();
                int pageSize = pageObj.getPageSize();
                int pageNum = pageObj.getPageNum();


                BaseUtils.loggerDebug("总页数:" + pages + "  总记录数:" + total + "  每页显示条数:" + pageSize + "  当前页:" + pageNum);
                BaseUtils.loggerDebug("Column:" + pageObj.getCountColumn());
                BaseUtils.loggerDebug("StartRow:" + pageObj.getStartRow());//开始行
                BaseUtils.loggerDebug("EndRow:" + pageObj.getEndRow());//结束的数据行
                BaseUtils.loggerDebug("PageNum:" + pageObj.getPageNum());//当前页


                resultMap.put("sEcho", pages);
                resultMap.put("iTotalRecords", total);
                resultMap.put("iTotalDisplayRecords", pageSize);
                resultMap.put("aaData", pageObj.getResult());
            }

        } catch (Exception e) {
            e.getMessage();
            resultMap.put("msg", BaseUtils.loggerError(e));
        }
        return resultMap;
    }

    private Page<EmpModel> find(Integer page, Integer limit, String search, String searchCondition, String[] columns, String order) {
        Example example = new Example(EmpModel.class);

        //搜索框
        Example.Criteria criteria = example.createCriteria();
        if (StringUtils.isNotBlank(search)) {
            example.or().andLike("ename", "%" + search + "%");
            example.or().andLike("deptno", "%" + search + "%");
            example.or().andLike("eid", "%" + search + "%");
        }

        //自定义查询条件
        if (StringUtils.isNotBlank(searchCondition)) {
            Map<String, Object> map = JSONObject.parseObject(searchCondition, new TypeReference<Map<String, Object>>() {
            });
            if (map != null) {
                String ename = map.get("ename").toString();
                String deptno = map.get("deptno").toString();
                if (StringUtils.isNotBlank(ename)) {
                    criteria.andEqualTo("ename", ename);
                }
                if (StringUtils.isNotBlank(deptno)) {
                    criteria.andEqualTo("deptno", deptno);
                }
            }
        }

        //排序
        if (StringUtils.isNotBlank(order)&&!"[]".equals(order)) {
            BaseUtils.loggerDebug(order);
            Map<String, String> orderMap = JSONObject.parseObject(order, new TypeReference<Map<String, String>>() {
            });
            //[{"column":1,"dir":"asc"}]
            String column = orderMap.get("column");
            String dir = orderMap.get("dir");
            if (dir.equalsIgnoreCase("asc")) {
                example.orderBy(columns[Integer.parseInt(column)]).asc();
            }else {
                example.orderBy(columns[Integer.parseInt(column)]).desc();
            }
        }

        Page<EmpModel> pageObj = PageHelper.startPage(page, limit, true);
        empMapper.selectByExample(example);
        return pageObj;
    }
```

## 前台代码

### HTML

```html
 <table id="table1" class="table table-bordered table-hover">
     <thead>
         <tr>
             <th nowrap="nowrap">序号</th>
             <th nowrap="nowrap">测试名称</th>
             <th nowrap="nowrap">测试分组</th>
             <th nowrap="nowrap">时间规则</th>
             <th nowrap="nowrap">调用类路径</th>
             <th nowrap="nowrap">操作</th>
         </tr>
     </thead>
     <tr>
         <td></td>
         <td></td>
         <td></td>
         <td></td>
         <td></td>
         <td></td>
     </tr>
</table>
```

### JS

```js
var columns = [
    {
        "data": null,
        "render": function (data, type, full, meta) {
            return meta.row + 1 + meta.settings._iDisplayStart;
        }
    },
    {
        "data": "eid",
        "searchable": true,
        "title": "ID"
    },
    {
        "data": "ename",
        "searchable": true,
        "title": "名称"
    },
    {
        "data": "job",
        "searchable": true,
        "title": "工作"
    },
    {
        "data": "deptno",
        "searchable": true,
        "title": "部门"
    },
    {
        "data": "eid",
        "title": "操作",
        "render": function (data, type, full, meta) {
            var param = '<a href="#" onClick="updateOperation( ' + data + ')" class="btn btn-info btn-sm" data-placement="top" data-toggle="tooltip" title="修改"> <i class="fa fa-edit" aria-hidden="true"></i> 修 改</a> \n' +
                '<a id="btnDelete" href="#" onClick="DeleteOperation(' + data + ')" class="btn btn-info btn-sm" data-placement="top" data-toggle="tooltip" title="删除"> <i class="fa fa-trash-o" aria-hidden="true"></i> 删 除</a> \n' +
                '<a href="#" onClick="OpenDlg(\'查看详细\',\'divDetail\',' + data + ')" class="btn btn-info btn-sm" data-placement="top" data-toggle="tooltip" title="查看详细"> <i class="fa fa-file-image-o" aria-hidden="true"></i> 详细</a>';
            return param;
        }

    }
];
$(function () {

    tableBackEndPage("table1", path + "/emp/empList.do", columns);

});

//弹出div
function OpenDlg(dlgTitle, content, id) {

    alert(id);


    var dlgIndex = layer.open({
        type: 1,
        title: dlgTitle,
        shadeClose: true,
        shade: false,
        area: ['60%', '70%'],
        maxmin: true, //开启最大化最小化按钮
        content: $('#' + content + '')
    });

}

function updateOperation(id, isEnable) {
    alert("修改！");


    $("#txtEditStatus").val("update");
    OpenDlg('修改记录', 'divEdit');
}
function DeleteOperation(id, isEnable) {
    swal({
        title: "您确定要删除该记录吗?",
        text: "一旦删除，您将不能恢复该数据!",
        icon: "warning",
        buttons: true,
        dangerMode: true
    }).then(function (willDelete) {
        if (willDelete) {
            //根据id执行删除操作
            // if (isBlankObj(result.msg)) {
                swal("记录已删除!", {
                    icon: "success"
                });
           /*     initTable();
            } else {
                swal(result.msg, {
                    icon: "error"
                });
            }*/
        } else {
            swal("已取消删除操作!");
        }

    });
}
```

**utils.js**

```js

/**
 * 加载表格数据   后端分页
 * @param tableName        表格的Id
 * @param url              请求的URL
 * @param columns          表格的所有列  格式:[{},{}]
 * @param searchCondition  查询的条件
 * @param async  是否异步
 */
function tableBackEndPage(tableName, url, columns, searchCondition, async) {
    if (isBlankObj(tableName)) {
        layer.msg("请传入表格的Id!", {times: 1000});
        return;
    }
    if (isBlankObj(url)) {
        layer.msg("请传入请求的URL!", {times: 1000});
        return;
    }
    if (!isBlankObj(columns)) {
        if (!isJsonObject(columns)) {
            layer.msg("请传入正确的列!", {times: 1000});
            return;
        }
    } else {
        layer.msg("请传入列!", {times: 1000});
        return;
    }

    // $('#' + tableName).dataTable().fnClearTable();
    $('#' + tableName).dataTable().fnDestroy();

    async = isBlankObj(async) ? false : true;

    var col = "";
    for (var key in columns) {
        col += columns[key].data + "~";
    }

    var lang = {
        "sProcessing": "处理中...",
        "sLengthMenu": "每页 _MENU_ 项",
        "sZeroRecords": "没有匹配结果",
        "sInfo": "当前显示第 _START_ 至 _END_ 项，共 _TOTAL_ 项。",
        "sInfoEmpty": "当前显示第 0 至 0 项，共 0 项",
        "sInfoFiltered": "(由 _MAX_ 项结果过滤)",
        "sInfoPostFix": "",
        "sSearch": "搜索:",
        "sUrl": "",
        "sEmptyTable": "表中数据为空",
        "sLoadingRecords": "载入中...",
        "sInfoThousands": ",",
        "oPaginate": {
            "sFirst": "首页",
            "sPrevious": "上页",
            "sNext": "下页",
            "sLast": "末页",
            "sJump": "跳转"
        },
        "oAria": {
            "sSortAscending": ": 以升序排列此列",
            "sSortDescending": ": 以降序排列此列"
        }
    };
    //初始化表格
    $("#" + tableName).dataTable({
        language: lang,  //提示信息
        autoWidth: false,  //禁用自动调整列宽
        stripeClasses: ["odd", "even"],  //为奇偶行加上样式，兼容不支持CSS伪类的场合
        processing: true,  //隐藏加载提示,自行处理
        serverSide: true,  //启用服务器端分页
        searching: true,  //禁用原生搜索
        orderMulti: false,  //启用多列排序
        order: [],  //取消默认排序查询,否则复选框一列会出现小箭头
        renderer: "bootstrap",  //渲染样式：Bootstrap和jquery-ui
        pagingType: "simple_numbers",  //分页样式：simple,simple_numbers,full,full_numbers
        columnDefs: [{
            "targets": 'nosort',  //列的样式名
            "orderable": true    //包含上样式名‘nosort’的禁止排序
        }],
        ajax: function (data, callback, settings) {
            //封装请求参数
            console.log("请求参数：");
            console.log(data);
            var param = {};
            param.limit = data.length;//页面显示记录条数，在页面显示每页显示多少项的时候
            param.start = data.start;//开始的记录序号
            param.page = data.start / data.length;//当前页码 从零开始
            // param.order = data.order[1];
            param.order = JSON.stringify(data.order[0]);
            console.log(JSON.stringify(data.order));
            param.search = data.search;
            //所有的列名
            param.col = col;

            if (!isBlankObj(searchCondition)) {
                console.log("searchCondition:" + JSON.stringify(searchCondition));
                param.searchCondition = JSON.stringify(searchCondition);
            }
            // alert(JSON.stringify(param));
            console.log(param);
            //console.log(param);
            //ajax请求数据
            $.ajax({
                type: "POST",
                url: url,
                cache: false,  //禁用缓存
                data: param,  //传入组装的参数
                async: async,//异步请求
                dataType: "json",
                success: function (result) {
                    console.log(result);
                    //setTimeout仅为测试延迟效果
                    // setTimeout(function () {
                    //封装返回数据
                    var returnData = {};
                    returnData.draw = data.draw;//这里直接自行返回了draw计数器,应该由后台返回
                    // returnData.recordsTotal = result.total;//返回数据全部记录
                    returnData.recordsTotal = result.iTotalRecords;//返回数据全部记录
                    // returnData.recordsFiltered = result.total;//后台不实现过滤功能，每次查询均视作全部结果
                    returnData.recordsFiltered = result.iTotalRecords;//后台不实现过滤功能，每次查询均视作全部结果
                    returnData.data = result.aaData;//返回的数据列表
                    //console.log(returnData);
                    //调用DataTables提供的callback方法，代表数据已封装完成并传回DataTables进行渲染
                    //此时的数据需确保正确无误，异常判断应在执行此回调前自行处理完毕
                    callback(returnData);
                    // }, 200);
                }
            });
        },
        //列表表头字段
        columns: columns
    }).api();//此处需调用api()方法,否则返回的是JQuery对象而不是DataTables的API对象
}

/**
 * 判断是否是JSON字符串
 * @param str
 * @returns {boolean}
 */
function isJsonString(str) {
    try {
        if (typeof JSON.parse(str) == "object") {
            return true;
        }
    } catch(e) {
    }
    return false;
}

/**
 * 判断对象是否是JSON数组
 * @param str
 */
function isJsonObject(str) {
    if(isBlankObj(str)){
        return false;
    }
    for (var key in str) {
        if (!isJsonString(JSON.stringify(str[key]))) {
            return false;
        }
    }
    return true;
}

function isBlankObj(obj){
    if(typeof(obj) == "boolean"){ // 布尔类型判断
        return false;
    }
    if(typeof(obj) == "number"){ // 处理数字为0时的判断
        return false;
    }
    if(isJSON(obj) && isEmptyObject(obj)){ // TODO 判断Json是否空
        return true;
    }
    if(obj == "" || obj == undefined || obj == null){
        return true;
    }
    if (typeof(obj) == "string" && obj.replace(/(^\s*)|(\s*$)/g, "").length == 0){ // 过滤空格，制表符，换页符
        return true;
    }
    return false;
}

/**
 * 判断是否是JSON对象
 * @param obj
 * @returns {boolean}
 */
function isJSON(obj){
    return typeof(obj) == "object" && Object.prototype.toString.call(obj).toLowerCase() == "[object   object]" && !obj.length;
}


/**
 * 判断JSON对象是否为空
 * @param obj
 * @returns {boolean}
 */
function isEmptyObject(obj) {
    for ( var name in obj ) {
        return false;
    }
    return true;
}

```



## 相关属性

**Columns说明**

虽然我们可以通过DOM直接获取DataTables元素的信息，但是DataTables提供了更方便的方法，可以自定义列的属性。下边就让我们一起来学习DataTables是怎么来定义列属性的。

- DataTables提供了两个参数来定义列属性：columns 和 columnDefs (源代码里：aoColumns 和 aoColumnDefs)
- 为了用户定义的参数更易于理解，DataTables提供的用户参数名和源代码的参数名是不一样的，不过这两个参数名，不管使用哪个，最终效果是一样的。（＊以下参数说明都是用户使用参数名）

**columns 和 columnDefs的区别：**

- 相同点：达到相同的效果
- 不同点：作用不一样，使用不一样（需要一个目标属性在每个定义的对象（columnDefs.targetsDT））
- columns：设置特定列的初始化属性，可以定义数组设置多列，数组长度必须等于表格的数量，只想使用默认值可以设为“NULL”，数组每个元素只能设置单列的属性。
- columnDefs：与columns非常相似，该数组可以针对特定的列，多列或者所有列定义。数组可以任意长度。通过targets参数设置一个列或者多列，该属性定义可以如下：
  - 0或正整数 - 从左边的列索引计数 
  - 负整数 - 列索引从右边计数
  - 一个字符串 - 类名称将被匹配上的TH为列 
  - 字符串“_all” - 所有的列（即指定一个默认值）
- 两个参数可以同时使用，但是columns定义的优先级最高。
- 当columnDefs里对同一列有多个定义时，最开始的定义优先级最高。

```js
 $('#example').dataTable(  
        {  
            data: [  
                    {  
                        "name":    "Tiger Nixon1",  
                        "position":   "System Architect1",  
                        "phone": { "plain": 5552368, "filter": "5552368 555-2368", "display": "555-2368" },  
                        "salary":    "$3,1201",  
                        "start_date": "2011/04/25",  
                        "office":    "Edinburgh1",  
                        "extn":    "54211"  
                    },  
                    {  
                        "name":    "Tiger Nixon2",  
                        "position":   "System Architect2",  
                        "phone": { "plain": 5552368, "filter": "5552368 555-2368", "display": "555-2368" },  
                        "salary":    "$3,1202",  
                        "start_date": "2011/04/25",  
                        "office":    "Edinburgh2",  
                        "extn":    "54212"  
                    },  
                    {  
                        "name":    "Tiger Nixon3",  
                        "position":   "System Architect3",  
                        "phone": { "plain": 5552368, "filter": "5552368 555-2368", "display": "555-2368" },  
                        "salary":    "$3,1203",  
                        "start_date": "2011/04/25",  
                        "office":    "Edinburgh3",  
                        "extn":    "54213"  
                    }  
                      
            ],  
              
            columnDefs: [  
                {  
                    "targets": 0,  
                    "searchable": false  
                },  
                {  
                    "targets": [1,2,3],  
                    "orderData": [ 2, 3, 4 ],  
                    "searchable": false  
                },  
                {  
                    "targets": [-3,-4],  
                    "orderable": false,  
                    "searchable": false  
                }  
            ],  
              
            columns: [  
                { "name": "name",   
                  "cellType": "th",  
                  "orderDataType": "dom-text",  
                  "orderSequence": [ "desc","asc", "asc" ],  
                  "className": "my_class",  
                  "contentPadding": "mmm",  
                  "createdCell": function (td, cellData, rowData, row, col) {  
                      if ( row < 1 ) {  
                        $(td).css('color', 'red');  
                      }  
                    },  
                  "data": "name",   
                  "searchable": true,   
                  "title": "My Name"  
                },  
                {   
                    "data": "position",  
                    "render": function ( data, type, full, meta ) {  
                        return '<a href="'+data+'">' + data + '</a>';  
                    }  
                },  
                {  
                    "data": 'phone',  
                    "render": {  
                      "_": "plain",  
                      "filter": "filter",  
                      "display": "display"  
                    }  
                },  
                { "data": "office" },  
                { "data": "start_date", "type": "date" },  
                { "data": "extn", "visible": false},  
                { "data": "salary", "width": "20px"  },  
                {  
                    "data": null,  
                    "orderable": false,  
                    "defaultContent": "<button>Edit</button>"  
                }  
      
            ]  
        }  
    );
```

**参数详解：**

|   用户参数名   |   源码参数名    |                           英文解释                           |                           中文解释                           |
| :------------: | :-------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|    cellType    |    sCellType    |             Cell type to be created for a column             |                设置列标签的类型（ex：th，td）                |
|   className    |     sClass      |          Class to assign to each cell in the column          |                     设置列的class属性值                      |
| contentPadding | sContentPadding | Add padding to the text content used when calculating the optimal with for a table. | 设置填充内容，以计算与优化为一个表时所使用的文本内容，一般不需要设置 |
|  createdCell   |  fnCreatedCell  |       Cell created callback to allow DOM manipulation        |       设置cell创建完后的回调函数，设置背景色或者添加行       |
|      data      |      mData      | Set the data source for the column from the rows data object / array |                       设置单元格里的值                       |
| defaultContent | sDefaultContent |          Set default, static, content for a column           |                        设置列的默认值                        |
|      name      |      sName      |             Set a descriptive name for a column              |                      设置列的描述性名称                      |
|   orderable    |    bSortable    |          Enable or disable ordering on this column           |                      设置列是否可以排序                      |
|   orderData    |    aDataSort    | Define multiple column ordering as the default order for a column |                  设置多列排序时列的默认顺序                  |
| orderDataType  |  sSortDataType  |               Live DOM sorting type assignment               |                                                              |
| orderSequence  |    asSorting    |             Order direction application sequence             |          设置列的默认排序，可以改变列排序的顺序处理          |
|     render     |     mRender     |        Render (process) the data for use in the table        |                                                              |
|   searchable   |   bSearchable   |    Enable or disable filtering on the data in this column    |                     设置列的数据是否过滤                     |
|     title      |     sTitle      |                     Set the column title                     |                         设置列的标题                         |
|      type      |      sType      | Set the column type - used for filtering and sorting string processing.Four types (string, numeric, date and html (which will strip HTML tags before ordering)) are currently available. |          设置列的类型，用于过滤和排序的字符串处理。          |
|    visible     |    bVisible     |         Enable or disable the display of this column         |                        设置列是否显示                        |
|     width      |     sWidth      |                   Column width assignment                    |                         定义列的宽度                         |