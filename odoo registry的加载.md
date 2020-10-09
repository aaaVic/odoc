# 参考：http://buke.github.io/blog/2013/02/26/openerp-dynamic-loading-and-booting-way/



1. odoo/13.0/odoo/cli/server.py: main()
```
rc = odoo.service.server.start(preload=preload, stop=stop)
```

2. odoo/13.0/odoo/service/server.py: start()
```
if odoo.evented:
  server = GeventServer(odoo.service.wsgi_server.application)
elif config['workers']:
  if config['test_enable'] or config['test_file']:
  _logger.warning("Unit testing in workers mode could fail; use --workers 0.")

server = PreforkServer(odoo.service.wsgi_server.application)

# Workaround for Python issue24291, fixed in 3.6 (see Python issue26721)
if sys.version_info[:2] == (3,5):
  # turn on buffering also for wfile, to avoid partial writes (Default buffer = 8k)
  werkzeug.serving.WSGIRequestHandler.wbufsize = -1
else:
  server = ThreadedServer(odoo.service.wsgi_server.application)
 
......

rc = server.run(preload, stop)
```

3. odoo/13.0/odoo/service/server.py: （Gevent/Prefork/Threaded）Server.run()
```
rc = preload_registries(preload)
```

4. odoo/13.0/odoo/service/server.py: preload_registries()
```
registry = Registry.new(dbname, update_module=update_module)
```

5. odoo/13.0/odoo/modules/registry.py: Registry.new()
  根据数据库名，加载/新建registry，除了启动还有以下几种情况会执行：
  - _initialize_db（exp_create_database）: 新建数据库
  - exp_duplicate_database: 复制数据库
  - restore_db: 恢复数据库
  - exp_migrate_databases: 迁移数据库
  - _button_immediate_function: 模块安装
  - upgrade_module（load_modules.update）: 更新模块
  - import_translation: 导入翻译，执行main时，translate_out
  - export_translation: 导出翻译，执行main时，translate_in
  - check_signaling: 检查registry是否发生变化，发现registry发生变化时

6. odoo/13.0/odoo/modules/loading.py: load_modules
  - 加载base模块
  ```
  graph = odoo.modules.graph.Graph()
  graph.add_module(cr, 'base', force)
  
  report = registry._assertion_report
  loaded_modules, processed_modules = load_module_graph(cr, graph, status, perform_checks=update_module,report=report, models_to_check=models_to_check)
  ```
  - 加载（install/update）其他模块
  - 加载已安装的模块/安装处于待安装状态的模块: load_marked_modules load_module_graph
  - migration
  - 清理升级时，不再需要的record
  ```
  env['ir.model.data']._process_end(processed_modules)
  env['base'].flush()
  ```
  - 卸载to remove的模块

7. odoo/13.0/odoo/modules/loading.py: load_module_graph
  - load_openerp_module(package.name): import 模块

  - 如果是安装模块，会执行pre_init
  ```
  if new_install:
  py_module = sys.modules['odoo.addons.%s' % (module_name,)]
  pre_init = package.info.get('pre_init_hook')
  if pre_init:
  getattr(py_module, pre_init)(cr)
  ```
  - model_names = registry.load(cr, package): 调用odoo元类的初始化（module_to_models）, 通过_build_model实例化model(处理继承),
实例化后通过descendants更新registry.models，加载model

  - 升级模块：
  ```
  load_data
  load_demo
  ```
  - 如果安装模块，会执行post_init：
  ```
  if new_install:
  post_init = package.info.get('post_init_hook')
  if post_init:
  getattr(py_module, post_init)(cr, registry)
  ```

