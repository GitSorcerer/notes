# DataTable

# 1.相关设置

设置复选框

```json
{
	"sTitle": '<label><input id="selectAll" type=\'checkbox\' name=\'checkSelect\' onclick="selectAll()" /><span class=\'lbl\'></span></label>',
	"sClass": "center",
	"bSortable": false,
	"sWidth": "20",
	"mRender": function(settings, rowIdx, rec, type) {
		var date = rec.eid + "/" + rec.ename;
		var btnBind = "<label><input type='checkbox' name='checkBox1' value='" + date + "'/><span class='lbl'></span></label>";
		return btnBind;
	}
}
```

回选

```js
function selectAll() {
    //id=all是全选/反选的checkbox，判断是选中还是没选中
    var isChecked = $("#selectAll").is(':checked');
    //获取所有checkbox组成的数组
    var checkedArrs = $("input[type='checkbox']:checked");
    //判断是全选还是反选
    if (isChecked) {
        $("input[type='checkbox']").each(function () {
            $(this).prop('checked', true);
        });
    } else {
        $("input[type='checkbox']").each(function () {
            $(this).removeAttr('checked', false);
        });
    }
}
```

