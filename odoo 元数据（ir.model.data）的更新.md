- py(ir.model/ir.model.fields/ir.model.fields.selection/ir.model.constraint):
  - 数据变更时，调用init_models
  - 调用_auto_init, 在post_init中注册一个函数_reflect，并执行
  - 分别调用_reflect_model
  - 调用_update_xmlids更新数据


- xml(ir.ui.view/ir.ui.menu/ir.actions.act_window...)
  - load_module_graph时，执行load_data(cr, idref, mode, kind='data', package=package, report=report)加载/更新manifest中data的配置
  - 解析xml: tools.convert_file(cr, package.name, filename, idref, mode, noupdate, kind, report), 因为文件是xml, 所以走进convert_xml_import 判断分支
  - 根据RNG文件解析xml, 初始化xml_import, 并对解析后的xml调用parse
  - 对xml中根目录下的记录进行解析，调用_tag_root， 然后通过不同的标签，调用不同的解析方法（_tag_record/_tag_delete/_tag_function...）
    如果xml的记录层级为:
    ```xml
    <odoo>
      <data>
        <record/>
        ....
      </data>
    </odoo>
    ```
    拿到的根目录是odoo，odoo下的标签是data，则会再调用_tag_root
  - 然后会调用 odoo/13.0/odoo/models.py: _load_records，创建/更新到对应模型中(ir.ui.view/ir.ui.menu/ir.actions.act_window...)
  - 最后调用ir.model.data中的_update_xmlids更新元数据
