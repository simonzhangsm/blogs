---
title: wecenter学习笔记
date: 2016-08-12 11:08:01
tags: [php, wecenter, 笔记]
---

> wecenter是一个轻量级的问答社区的开源应用
使用私有的授权协议，商业用途必须付费才能使用，个人非商业用途无需授权。
官方主页： http://wecenter.com

在学习过程中，存下该笔记，仅参考其实现方法和原理，如需直接使用wecenter涉及版权部分源码请获取授权后再使用。

## 模版视图渲染框架Savant3

核心代码如下：

```
public function fetch($tpl = null)
	{
		// make sure we have a template source to work with
		if (is_null($tpl)) {
			$tpl = $this->__config['template'];
		}
		
		// get a path to the compiled template script
		$result = $this->template($tpl);
		
		// did we get a path?
		if (! $result || $this->isError($result)) {
		
			// no. return the error result.
			return $result;
			
		} else {
		
			// yes.  execute the template script.  move the script-path
			// out of the local scope, then clean up the local scope to
			// avoid variable name conflicts.
			$this->__config['fetch'] = $result;
			unset($result);
			unset($tpl);
			
			// are we doing extraction?
			if ($this->__config['extract']) {
				// pull variables into the local scope.
				extract(get_object_vars($this), EXTR_REFS);
			}
			
			// buffer output so we can return it instead of displaying.
			ob_start();
			
			// are we using filters?
			if ($this->__config['filters']) {
				// use a second buffer to apply filters. we used to set
				// the ob_start() filter callback, but that would
				// silence errors in the filters. Hendy Irawan provided
				// the next three lines as a "verbose" fix.
				ob_start();
				include $this->__config['fetch'];
				echo $this->applyFilters(ob_get_clean());
			} else {
				// no filters being used.
				include $this->__config['fetch'];
			}
			
			// reset the fetch script value, get the buffer, and return.
			$this->__config['fetch'] = null;
			return ob_get_clean();
		}
	}
```

分两步:

1. 将assign设置到Savant3实例的成员变量导入到当前符号表中
   
   ```extract(get_object_vars($this), EXTR_REFS);```
   
2. 执行模版
   ```include $this->__config['fetch'];```



## Zend Session 框架

### PHP runtime对Session的支持

* 启动新会话或者重用现有会话

```
bool session_start ([ array $options = [] ] )
```

* 使用新生成的会话 ID 更新现有会话 ID

```
bool session_regenerate_id ([ bool $delete_old_session = false ] )
```

*  设置用户自定义会话存储函数

```
bool session_set_save_handler ( callable $open , callable $close , callable $read , callable $write , callable $destroy , callable $gc [, callable $create_sid ] )
```

* 销毁一个会话中的全部数据

```
bool session_destroy ( void )
```

* 保存Session数据并结束Session

```
void session_write_close ( void )
```

* 设置Cookie

```
bool setcookie ( string $name [, string $value = "" [, int $expire = 0 [, string $path = "" [, string $domain = "" [, bool $secure = false [, bool $httponly = false ]]]]]] )
```

* 其它

[PHP手册－函数参考－Session 函数](http://php.net/manual/zh/ref.session.php)

### 什么是Session命名空间

Zend使用命名空间来隔离不同的Session数据，对应类`Zend_Session_Namespace`。

数据仍然存放在全局变量 $_SESSION中，不同的空间的数据存放到如下的键下：

```
$_SESSION['namespace']
```

### Session数据持久化

Zend Session默认支持两种存储方式：

* 存储到文件
* 存储到数据库

Zend Session指定了规范化的持久化接口，包括：

* open
* close
* read
* write
* destroy
* gc

统一通过实现接口 `Zend_Session_SaveHandler_Interface` 来实现存储到分布式缓存。

序列化session数据到数据表中
```
Zend_Session::setSaveHandler(new Zend_Session_SaveHandler_DbTable(array(
			    'name' 					=> get_table('sessions'),
			    'primary'				=> 'id',
			    'modifiedColumn'		=> 'modified',
			    'dataColumn'			=> 'data',
			    'lifetimeColumn'		=> 'lifetime',
				//'authIdentityColumn'	=> 'uid'
			)));
```

表结构大抵如下：

```
CREATE TABLE `aws_sessions` (
  `id` varchar(32) NOT NULL COMMENT 'session id',
  `modified` int(10) NOT NULL COMMENT '修改时间',
  `data` text NOT NULL COMMENT 'Session数据',
  `lifetime` int(10) NOT NULL COMMENT '有效时间',
  PRIMARY KEY (`id`),
  KEY `modified` (`modified`),
  KEY `lifetime` (`lifetime`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;
```

字段名可以通过 `Zend_Session_SaveHandler_DbTable`的构造函数参数定制

### 创建和恢复Session

通过如下方式来启动Session

```
Zend_Session::start();

self::$session = new Zend_Session_Namespace(G_COOKIE_PREFIX . '_Anwsion');
```

### 终止Session和Session的有效期

一般不需要主动调用`writeClose`， Session数据会在脚本执行完自动保存。

Session的有效期可以通过修改`php.ini`的`[session]`

```
; After this number of seconds, stored data will be seen as 'garbage' and
; cleaned up by the garbage collection process.
; http://php.net/session.gc-maxlifetime
session.gc_maxlifetime = 1440
```

### rememberme

设置cookied的失效时间来实现remember-me

> system/Zend/Session.php#rememberUntil

```
$cookieParams = session_get_cookie_params();

session_set_cookie_params(
    $seconds,
    $cookieParams['path'],
    $cookieParams['domain'],
    $cookieParams['secure']
    );
```

### 用户登陆判断

通过解密`_user_login`的cookie获取用户登陆信息，来验证是否登陆

> system/core/user.php#get_info

```
		if (! AWS_APP::session()->client_info AND $_COOKIE[G_COOKIE_PREFIX . '_user_login'])
		{
			$auth_hash_key = md5(G_COOKIE_HASH_KEY . $_SERVER['HTTP_USER_AGENT']);

			// 解码 Cookie
			$sso_user_login = json_decode(AWS_APP::crypt()->decode($_COOKIE[G_COOKIE_PREFIX . '_user_login'], $auth_hash_key), true);

			if ($sso_user_login['user_name'] AND $sso_user_login['password'] AND $sso_user_login['uid'])
			{
				if ($user_info = AWS_APP::model('account')->check_hash_login($sso_user_login['user_name'], $sso_user_login['password']))
				{
					AWS_APP::session()->client_info['__CLIENT_UID'] = $user_info['uid'];
					AWS_APP::session()->client_info['__CLIENT_USER_NAME'] = $user_info['user_name'];
					AWS_APP::session()->client_info['__CLIENT_PASSWORD'] = $sso_user_login['password'];

					return true;
				}
			}

			HTTP::set_cookie('_user_login', '', null, '/', null, false, true);

			return false;
		}
```

也就是说，判断用户是否登陆过依赖的并不是session，而是存储的cookie中的用户信息。如果用户密码或用户名修改了，登陆信息也会失效，符合设计要求。

登陆成功后还会将登陆的用户信息存入Session Data（最终会系列化存储），如果Session有效，即使_user_login的Cookied无效，用户也可算是已登陆的。

### URL改写

通常，wecenter的地址如下：

```
http://host/?/article/8?id=1&wtf=other
```
其中IndexScript（`?/`）部分可以修改

> system/config.inc.php

```
25 define('G_INDEX_SCRIPT', '?/');
```

中间部分称为`动作`，格式如 `/模块名/控制器/动作/ID`，具体规则为

* 如果使用 /模块名/控制器/动作/ID 格式 Query string 的使用可以参照 兼容性的支持

* 如果动作在 main 控制器中可以省略, 例: account/main/login/ 等同于 account/login/

* 如果动作名为 index 可以省略,  例: account/login/index/ 等同于 account/login/

query string参数也可以通过规则改写

WeCenter 的查询字符串为使用 __ 分隔参数, 使用 – 为参数赋值, 在程序中直接使用 $_GET 取出内容
常规的: 

```
account/login/?return_url=1&callback=2
```

WeCenter 的: 

```
account/login/return_url-1__callback-2
```

兼容性支持

下面的几种 URL 形式在程序中都是被支持的:

http://domian/index.php?/question/id-320__column-log__source-doc

http://domian/index.php?/question/320?column=log&source=doc

http://domian/index.php?/question/?id=320&column=log&source=doc

http://domian/index.php?/question/320?column-log__source-doc

http://domian/index.php?/question/320&column-log__source-doc

> 注意：index.php是唯一的入口，action只能通过query string传入，并通过自定义的路由规则实现url到action的映射，具体rewrite的实现参照
> 
> system/core/url.php
>

URL最终被解析为

* app_dir
* controller
* action
* $_GET

## action路由

** 创建controller **

> system/aws_app.inc.php

```
public static function create_controller($controller, $app_dir)
	{
		if ($app_dir == '' OR trim($controller, '/') === '')
		{
			return false;
		}

		$class_file = $app_dir . $controller . '.php';

		$controller_class = str_replace('/', '_', $controller);

		if (! file_exists($class_file))
		{
			return false;
		}

		if (! class_exists($controller_class, false))
		{
			require_once $class_file;
		}

		// 解析路由查询参数
		load_class('core_uri')->parse_args();

		if (class_exists($controller_class, false))
		{
			return new $controller_class();
		}

		return false;
	}
```

** 执行action method **

> system/aws_app.inc.php#run

```
$action_method = load_class('core_uri')->action . '_action';

// 判断
if (! is_object($handle_controller) OR ! method_exists($handle_controller, $action_method))
{
	HTTP::error_404();
}

...

// 执行
if (!$_GET['id'] AND method_exists($handle_controller, load_class('core_uri')->action . '_square_action'))
{
    $action_method = load_class('core_uri')->action . '_square_action';
}

$handle_controller->$action_method();
```

## Controller访问控制实现原理

通过在Controller的`get_access_rule`来返回访问控制权限

```
public function get_access_rule()
{
	$rule_action['rule_type'] = 'white';

	$rule_action['actions'][] = 'index';
	$rule_action['actions'][] = 'save_comment';

	return $rule_action;
}
```

应用启动时会检查action是否在白名单或者黑名单中，从而引导未登陆的用户进行登陆

```
		// 判断访问规则使用白名单还是黑名单, 默认使用黑名单
		if ($access_rule)
		{
			// 黑名单, 黑名单中的检查 'white' 白名单,白名单以外的检查 (默认是黑名单检查)
			if (isset($access_rule['rule_type']) AND $access_rule['rule_type'] == 'white')
			{
				if ((! $access_rule['actions']) OR (! in_array(load_class('core_uri')->action, $access_rule['actions'])))
				{
					self::login();
				}
			}
			else if (isset($access_rule['actions']) AND in_array(load_class('core_uri')->action, $access_rule['actions']))	// 非白就是黑名单
			{
				self::login();
			}

		}
		else
		{
			self::login();
		}
```

白名单中的不需要登陆，黑名单中的action需要登陆

## Model读写分离

所有的具体的Model都从AWS_MODEL派生。

AWS_MODEL依赖core_db对Zend_Db的包装来支持数据库相关的功能，参照：

> system/core/db.php

初始化

```
$this->db['master'] = Zend_Db::factory(load_class('core_config')->get('database')->driver, load_class('core_config')->get('database')->master);
```

切换数据库

```
public function setObject($db_object_name = 'master')
{
	if (isset($this->db[$db_object_name]))
	{
		Zend_Registry::set('dbAdapter', $this->db[$db_object_name]);
		Zend_Db_Table_Abstract::setDefaultAdapter($this->db[$db_object_name]);

		$this->current_db_object = $db_object_name;

		return $this->db[$db_object_name];
	}

	throw new Zend_Exception('Can\'t find this db object: ' . $db_object_name);
}
```

AWS_MODEL提供如下功能

* 自动添加表前缀
* 数据库主从切换
* 基本CRUD
* 事务处理
* 脚本执行完后的数据库更新操作
* 分页查询
* 基本的统计 count\max\min\sum
* quote

通过对db操作的封装，方便将所有的查询操作切换到从库，更新使用主库，从而实现读写分离对具体Model的实现者透明。

## 自动引入机制和Autoload

通过约束model类、system类和config的存放位置和命名规则，实现了不需要事先引入文件就可以直接使用, 这使得在编程过程中变得方便快捷, 也避免了类库重复实例化的问题。

具体的规则如下：

1. model

	放在 model 目录下, 文件名: name.inc.php，类名采用name_class的格式
	* 在程序中使用方法: 
	
	```
	$this->model(‘name’)->action();
	```
	
	* 可用范围: CONTROLLER, Model
2. system

	放在 system 目录之下, 类名相对于 system 目录, 将 / 换成 _，例: 
	
	```
	Zend_Mail
	路径: system/Zend/Mail.php
	类名: Zend_Mail
	```
	
	* 在程序中使用方法: new, 静态调用, load_class(‘class_name’);
	* 可用范围: 任意, 不需要带参数实例化建议使用 load_class
3. config

	放在 system/config 目录之下, 文件内容为一个 $config 数组, 命名为 配置名.php
	* 在程序中使用方法: AWS_APP::config()->get(‘配置名’)->数组下标
	* 可用范围: 任意, 不需要带参数实例化建议使用 load_class

** Autoload **

在初始化脚本 `init.php` 里加载类

> system/init.php

```
128 load_class('core_autoload');
```

在 core_autoload的构造过程注册加载处理函数

> system/core/autoload.php#__construct

```
36 spl_autoload_register(array($this, 'loader'));
```

类加载函数处理

* 按路径加载
* 别名类的加载，包括：TPL/FORMAT/HTTP/H/ACTION_LOG
* models类的加载
* class类的加载

找到目标类的路径后直接加载，并缓存

```
require $class_file_location;

self::$loaded_class[$class_name] = $class_file_location;
```

plugin的model类加载

```
if (class_exists('AWS_APP', false))
{
	if (AWS_APP::plugins()->model())
	{
		self::$aliases = array_merge(self::$aliases, AWS_APP::plugins()->model());
	}
}
```

## 插件机制

虽然wecenter有一个初级插件模型，但很遗憾并没有第三方基于这套机制去开发插件，连官方站点的挂出的`插件`也只是采用二次开发的方式，并不是基于插件开发的。

### 目前有哪些插件

项目中内置两个插件

* aws_external
* aws_offical_external

提供了几个utility函数

### 插件的结构

典型的插件结构如下所示：

每个插件一个为plugins里的一个目录，目录下的`config.php`为插件的元数据信息

```
$aws_plugin = array(
	'title' => 'External for Anwsion',	// 插件标题
	'version' => 20130107,	// 插件版本
	'description' => 'Anwsion 官网调用插件',	// 插件描述
	'requirements' => '20120706',	// 最低 Build 要求
	
	'contents' => array(
		// 对控制器构造部署代码 (setup)
		'setups' => array(
			array(
			'app' => 'app',
			'controller' => 'controller',
			'include' => 'aws_plungin_controller.php'
			)
		),
	
		// 对控制器 Action 部署代码 (只支持模板输出之前)
		'actions' => array(
			array(
			'app' => 'app',
			'controller' => 'controller',
			'action' => 'action',
			'template' => 'template.tpl',
			'include' => 'aws_plungin_controller.php'
			)
		),
		
		// 注册 Model, 用 $this->model('name') 访问
		'model' => array(
			'class_name' => 'aws_offical_external_class',	// Model name, 以 _class 结尾
			'include' => 'aws_offical_external.php',	// 引入代码文件位置
		),
	),
);
```

包括：

* 初始化代码
* action
* model

### 插件的元数据加载

应用启动时加载插件管理器(core_plugins)

> aws_app.inc.app#init

```
117 self::$plugins = load_class('core_plugins');
```

检索所有插件目录，并加载元数据

> aws_app.inc.app#load_plugins()

```
$dir_handle = opendir($this->plugins_path);

	    while (($file = readdir($dir_handle)) !== false)
	    {
	       	if ($file != '.' AND $file != '..' AND is_dir($this->plugins_path . $file))
	       	{
	       		$config_file = $this->plugins_path . $file . '/config.php';

	            if (file_exists($config_file))
	            {
	            	$aws_plugin = false;

		            require_once($config_file);

		            if (is_array($aws_plugin) AND G_VERSION_BUILD >= $aws_plugin['requirements'])
		            {
		            	if ($aws_plugin['contents']['model'])
		            	{
			            	$this->plugins_model[$aws_plugin['contents']['model']['class_name']] = $this->plugins_path . $file . '/' . $aws_plugin['contents']['model']['include'];
		            	}

		            	if ($aws_plugin['contents']['setups'])
		            	{
			            	foreach ($aws_plugin['contents']['setups'] AS $key => $data)
			            	{
			            		if ($data['app'] AND $data['controller'] AND $data['include'])
			            		{
				            		$this->plugins_table[$data['app']][$data['controller']]['setup'][] = array(
				            			'file' => $this->plugins_path . $file . '/' . $data['include'],
				            		);
			            		}
			            	}
		            	}

		            	if ($aws_plugin['contents']['actions'])
		            	{
			            	foreach ($aws_plugin['contents']['actions'] AS $key => $data)
			            	{
			            		if ($data['app'] AND $data['controller'] AND $data['include'])
			            		{
				            		$this->plugins_table[$data['app']][$data['controller']][$data['action']][] = array(
				            			'file' => $this->plugins_path . $file . '/' . $data['include'],
				            			'template' => $data['template']
				            		);
			            		}
			            	}
		            	}

			            $this->plugins[$file] = $aws_plugin;
		            }
	            }
	        }
	    }

	   	closedir($dir_handle);
```

将元数据写入缓存

### 载入插件

Controller的初始化函数里会根据参数加载对应的插件

> aws_controller.inc.php#__construct

```
// 载入插件
if ($plugins = AWS_APP::plugins()->parse($_GET['app'], $_GET['c'], 'setup'))
{
	foreach ($plugins as $plugin_file)
	{
		include $plugin_file;
	}
}
```

模版渲染时会唤起插件

> system/class/cls_template.inc.php#output

```
if ($plugins = AWS_APP::plugins()->parse($_GET['app'], $_GET['c'], $_GET['act'], str_replace(self::$template_ext, '', $template_filename)))
{
	foreach ($plugins AS $plugin_file)
	{
		include_once $plugin_file;
	}
}
```

此外，定时任务也会唤起插件

> app/crond/main.php#run_action

```
if ($plugins = AWS_APP::plugins()->parse('crond', 'main', $call_action))
{
	foreach ($plugins AS $plugin_file)
	{
		include($plugin_file);
	}
}
```

### 插件怎么使用

由于Autoload会处理Plugin里model类的加载，所有model类的使用方式并无二般。如：

```
$this->model('aws_external')->format_js_question_ul_output($_GET['ul_class'],$_GET['is_recommend']));
```

插件的controller如何路由

## Cache的实现原理

wecenter支持两种缓存

* Memcached
* File

基于Zend_Cache实现的文件缓存机制。

> system/core/cache.php 

```
$this->cache_factory = Zend_Cache::factory($this->frontendName, $this->backendName, $this->frontendOptions, $this->backendOptions);
```

读缓存

```
$result = $this->cache_factory->load($this->cachePrefix . $key);
```

写缓存

```
$result = $this->cache_factory->save($value, $this->cachePrefix . $key, array(), $lifetime);
```

删除缓存

```
return $this->cache_factory->remove($key);
```

清理缓存

```
return $this->cache_factory->clean(Zend_Cache::CLEANING_MODE_ALL);
```

### Zend Cache

为了方便使用，Zend Cache根据缓存的数据类型构造了各种 FrontCache，以及根据缓存的存储类型引入了BackendCache的概念。

** Frontend **

* Capture
  
  Capture能将php的start和flush之间的输出缓存起来，内部实现挂了使用了ob_start主持的output_callback来缓存数据
  
  > Zend/Cache/Frontend/Capture.php#start
  
  ```
  ob_start(array($this, '_flush'));
  ob_implicit_flush(false);
  ```
  
  通过调用`ob_flush`来刷新缓冲区

* Page

  捕获输出的部分内容

* Output

	使用方式
	
	```
	// if it is a cache miss, output buffering is triggered
	if (! $cache->start('mypage')) {
	   // output everything as usual
	   echo 'Hello world! ';
	   echo 'This is cached ('.time().') ';
	   $cache->end(); // output buffering ends
	}
	echo 'This is never cached ('.time().').';
	```

* Class
	
	通过实现__call，提供对目标对象的代理，并缓存函数调用的结果。
	
	```
	public function __call($name, $parameters)
    {
        $callback = array($this->_cachedEntity, $name);

        if (!is_callable($callback, false)) {
            Zend_Cache::throwException('Invalid callback');
        }

        $cacheBool1 = $this->_specificOptions['cache_by_default'];
        $cacheBool2 = in_array($name, $this->_specificOptions['cached_methods']);
        $cacheBool3 = in_array($name, $this->_specificOptions['non_cached_methods']);
        $cache      = (($cacheBool1 || $cacheBool2) && (!$cacheBool3));

        if (!$cache) {
            // We do not have not cache
            return call_user_func_array($callback, $parameters);
        }

        $id = $this->makeId($name, $parameters);
        if (($rs = $this->load($id)) && (array_key_exists(0, $rs))
            && (array_key_exists(1, $rs))
        ) {
            // A cache is available
            $output = $rs[0];
            $return = $rs[1];
        } else {
            // A cache is not available (or not valid for this frontend)
            ob_start();
            ob_implicit_flush(false);

            try {
                $return = call_user_func_array($callback, $parameters);
                $output = ob_get_clean();
                $data   = array($output, $return);

                $this->save(
                    $data, $id, $this->_tags, $this->_specificLifetime,
                    $this->_priority
                );
            } catch (Exception $e) {
                ob_end_clean();
                throw $e;
            }
        }

        echo $output;
        return $return;
    }
	```
* Function

	缓存函数调用的结果
	
	```
	function call($callback, array $parameters = array(), $tags = array(), $specificLifetime = false, $priority = 8)
	```

	调用这个函数，会跟进函数名称和参数生成缓存主键，如果缓存有效，则直接返回缓存结果，否则调用函数并进行缓存，与classs的__call实现基本一致。

* File

	根据文件的修改状态来判断缓存是否有效。


** Backend **

需要符合基本的接口

> Zend/Cache/Backend/Interface.php

```
public function setDirectives($directives);
public function load($id, $doNotTestCacheValidity = false);
public function test($id);
public function save($data, $id, $tags = array(), $specificLifetime = false);
public function remove($id);
public function clean($mode = Zend_Cache::CLEANING_MODE_ALL, $tags = array());
```

* File

	基于文件系统的缓存。
	
	每个缓存除了生存缓存文件外，还会生出一个文件名以`internal-metadatas---`开头的元数据文件，并依靠元数据文件来存储数据的基本信息：

	> Zend/Cache/Backend/File.php#touch
		
	```
	'hash' => $metadatas['hash'],
    'mtime' => time(),
    'expire' => $metadatas['expire'] + $extraLifetime,
    'tags' => $metadatas['tags']
	```
	
* Sqlite

	将数据和元数据存储到sqlite中
	
	> Zend/Cache/Backend/Sqlite.php#_buildStructure
	
	```
	    private function _buildStructure()
    {
        $this->_query('DROP INDEX tag_id_index');
        $this->_query('DROP INDEX tag_name_index');
        $this->_query('DROP INDEX cache_id_expire_index');
        $this->_query('DROP TABLE version');
        $this->_query('DROP TABLE cache');
        $this->_query('DROP TABLE tag');
        $this->_query('CREATE TABLE version (num INTEGER PRIMARY KEY)');
        $this->_query('CREATE TABLE cache (id TEXT PRIMARY KEY, content BLOB, lastModified INTEGER, expire INTEGER)');
        $this->_query('CREATE TABLE tag (name TEXT, id TEXT)');
        $this->_query('CREATE INDEX tag_id_index ON tag(id)');
        $this->_query('CREATE INDEX tag_name_index ON tag(name)');
        $this->_query('CREATE INDEX cache_id_expire_index ON cache(id, expire)');
        $this->_query('INSERT INTO version (num) VALUES (1)');
    }
	```

	顾名思义，cache存储缓存数据，tag存储标签, version存储缓存的版本号。

	每次执行操作之前都会调用 `_checkAndBuildStructure` 检查是否需要构建表结构
	
	
* memcached

	依赖`Memcache`客户端来提供服务。

* libmemcached

	依赖`Memcached`来提供服务

* xcache
	
	来自于lightd团队的开源缓存服务器

* Apc
* BlackHole
* Static
* TwoLevels
* WinCache

	wincache仅支持NTS(非线程安全版本)的PHP

* ZendServer Disk
* ZendServer Share memeory
* ZendPlatform

涉及的缓存服务太多，不能一一探究。

 
## 配置参数管理

wecenter的配置有两种：

* system_setting规范化的键值对，通过get_setting访问
* 由core_config管理的各类异构配置数据

### get_setting

** 使用方式 **

```
get_setting('weibo_msg_enabled')
```

** 实现 **

> system/functions.inc.php

```
function get_setting($varname = null, $permission_check = true)
{
	if (! class_exists('AWS_APP', false))
	{
		return false;
	}

	if ($settings = AWS_APP::$settings)
	{
		// AWS_APP::session()->permission 是指当前用户所在用户组的权限许可项，在 users_group 表中，你可以看到 permission 字段
		if ($permission_check AND $settings['upload_enable'] == 'Y')
		{
			if (AWS_APP::session())
			{
				if (!AWS_APP::session()->permission['upload_attach'])
				{
					$settings['upload_enable'] = 'N';
				}
			}
		}
	}

	if ($varname)
	{
		return $settings[$varname];
	}
	else
	{
		return $settings;
	}
}
```

AWS_APP::$settings是从数据库加载的内容

> system/aws_app.inc.php#init

```
self::$settings = self::model('setting')->get_settings();
```

> models/setting.php#get_settings

```
	public function get_settings()
	{
		if ($system_setting = $this->fetch_all('system_setting'))
		{
			foreach ($system_setting as $key => $val)
			{
				if ($val['value'])
				{
					$val['value'] = unserialize($val['value']);
				}

				$settings[$val['varname']] = $val['value'];
			}
		}

		return $settings;
	}
```

system_setting表中的数据有安装脚本插入，内容皆为键值对

> install/system_settings.sql

```
INSERT INTO `[#DB_PREFIX#]system_setting` (`varname`, `value`) VALUES
('site_name', 's:8:"WeCenter";'),
('description', 's:30:"WeCenter 社交化知识社区";'),
('keywords', 's:47:"WeCenter,知识社区,社交社区,问答社区";'),
('sensitive_words', 's:0:"";'),
('def_focus_uids', 's:1:"1";'),
('answer_edit_time', 's:2:"30";'),
('cache_level_high', 's:2:"60";'),
...
```

### core_config

** 使用方式 **

```
$admin_menu = (array)AWS_APP::config()->get('admin_menu');
```

** 实现 **

get的调用会从内存中查找，如果找不到最终会调用`load_config`

> system/core/config.php#get

```
function get($config_id)
{
	...

	if (isset($this->config[$config_id]))
	{
		return $this->config[$config_id];
	}
	else
	{
		return $this->load_config($config_id);
	}
}
```

load_config会从config目录加载指定Id为文件名的配置文件，并缓冲到内存中

> system/core/config.php#load_config

```
function load_config($config_id)
{
	if (substr($config_id, -10) == '.extension' OR !file_exists(AWS_PATH . 'config/' . $config_id . '.php'))
	{
		throw new Zend_Exception('The configuration file config/' . $config_id . '.php does not exist.');
	}

	include_once(AWS_PATH . 'config/' . $config_id . '.php');

	if (!is_array($config))
	{
		throw new Zend_Exception('Your config/' . $config_id . '.php file does not appear to contain a valid configuration array.');
	}

	$this->config[$config_id] = (object)$config;

	return $this->config[$config_id];
}
```

另外，core_config还提供了一个动态写配置到文件的功能

> system/core/config.php#set

```
public function set($config_id, $data)
{
	if (!$data || ! is_array($data))
	{
		throw new Zend_Exception('config data type error');
	}

	$content = "<?php\n\n";

	foreach($data as $key => $val)
	{
		if (is_array($val))
		{
			$content .= "\$config['{$key}'] = " . var_export($val, true) . ";";;
		}
		else if (is_bool($val))
		{
			$content .= "\$config['{$key}'] = " . ($val ? 'true' : 'false') . ";";
		}
		else
		{
			$content .= "\$config['{$key}'] = '" . addcslashes($val, "'") . "';";
		}

		$content .= "\r\n";
	}

	$config_path = AWS_PATH . 'config/' . $config_id . '.php';

	$fp = @fopen($config_path, "w");

	@chmod($config_path, 0777);

	$fwlen = @fwrite($fp, $content);

	@fclose($fp);

	return $fwlen;
}
```

## 验证码

验证需要考虑

* 验证码图片的生成、存储和访问
* 验证码的有效期管理
* 失效验证码的清理策略
* 验证有效性

基于 `Zend_Captcha_Image` 实现的验证码生成和管理功能，可以根据自己提供的字体文件生成自定义大小和位数的验证码，
并存储到单独的session命名空间中``。

> system/core/captcha.php#__construct

```
$this->captcha = new Zend_Captcha_Image(array(
	'font' => $this->get_font(),
	'imgdir' => $img_dir,
	'fontsize' => rand(20, 22),
	'width' => 100,
	'height' => 40,
	'wordlen' => 4,
	'session' => new Zend_Session_Namespace(G_COOKIE_PREFIX . '_Captcha'),
	'timeout' => 600
));

$this->captcha->setDotNoiseLevel(rand(3, 6));
$this->captcha->setLineNoiseLevel(rand(1, 2));
```

** 使用方式和实现原理 **

生成验证码

```
AWS_APP::captcha()->generate();
```

> system/core/captcha.php#generate

```
public function generate()
{
	$this->captcha->generate();

	HTTP::no_cache_header();

	readfile($this->captcha->getImgDir() . $this->captcha->getId() . $this->captcha->getSuffix());

	die;
}
```


检查验证码

```
if (!AWS_APP::captcha()->is_validate($_POST['seccode_verify']) AND get_setting('register_seccode') == 'Y')
{
	H::ajax_json_output(AWS_APP::RSM(null, -1, AWS_APP::lang()->_t('请填写正确的验证码')));
}
```

> system/core/captcha.php#is_validate

```
public function is_validate($validate_code, $generate_new = true)
{
	if (strtolower($this->captcha->getWord()) == strtolower($validate_code))
	{
		if ($generate_new)
		{
			$this->captcha->generate();
		}
		
		return true;
	}

	return false;
}
```

### Zend_Captcha_Image

** 生成验证码图片 **

> Zend/Captcha/Image.php

```
public function generate()
{
    $id = parent::generate();
    $tries = 5;
    // If there's already such file, try creating a new ID
    while($tries-- && file_exists($this->getImgDir() . $id . $this->getSuffix())) {
        $id = $this->_generateRandomId();
        $this->_setId($id);
    }
    $this->_generateImage($id, $this->getWord());

    if (mt_rand(1, $this->getGcFreq()) == 1) {
        $this->_gc();
    }
    return $id;
}
```

> Zend/Captcha/Image.php

```
protected function _generateImage($id, $word)
{
    if (!extension_loaded("gd")) {
        //require_once 'Zend/Captcha/Exception.php';
        throw new Zend_Captcha_Exception("Image CAPTCHA requires GD extension");
    }

    if (!function_exists("imagepng")) {
        //require_once 'Zend/Captcha/Exception.php';
        throw new Zend_Captcha_Exception("Image CAPTCHA requires PNG support");
    }

    if (!function_exists("imageftbbox")) {
        //require_once 'Zend/Captcha/Exception.php';
        throw new Zend_Captcha_Exception("Image CAPTCHA requires FT fonts support");
    }

    $font = $this->getFont();

    if (empty($font)) {
        //require_once 'Zend/Captcha/Exception.php';
        throw new Zend_Captcha_Exception("Image CAPTCHA requires font");
    }

    $w     = $this->getWidth();
    $h     = $this->getHeight();
    $fsize = $this->getFontSize();

    $img_file   = $this->getImgDir() . $id . $this->getSuffix();
    if(empty($this->_startImage)) {
        $img        = imagecreatetruecolor($w, $h);
    } else {
        $img = imagecreatefrompng($this->_startImage);
        if(!$img) {
            //require_once 'Zend/Captcha/Exception.php';
            throw new Zend_Captcha_Exception("Can not load start image");
        }
        $w = imagesx($img);
        $h = imagesy($img);
    }
    $text_color = imagecolorallocate($img, 0, 0, 0);
    $bg_color   = imagecolorallocate($img, 255, 255, 255);
    imagefilledrectangle($img, 0, 0, $w-1, $h-1, $bg_color);
    $textbox = imageftbbox($fsize, 0, $font, $word);
    $x = ($w - ($textbox[2] - $textbox[0])) / 2;
    $y = ($h - ($textbox[7] - $textbox[1])) / 2;
    imagefttext($img, $fsize, 0, $x, $y, $text_color, $font, $word);

   // generate noise
    for ($i=0; $i<$this->_dotNoiseLevel; $i++) {
       imagefilledellipse($img, mt_rand(0,$w), mt_rand(0,$h), 2, 2, $text_color);
    }
    for($i=0; $i<$this->_lineNoiseLevel; $i++) {
       imageline($img, mt_rand(0,$w), mt_rand(0,$h), mt_rand(0,$w), mt_rand(0,$h), $text_color);
    }

    // transformed image
    $img2     = imagecreatetruecolor($w, $h);
    $bg_color = imagecolorallocate($img2, 255, 255, 255);
    imagefilledrectangle($img2, 0, 0, $w-1, $h-1, $bg_color);
    // apply wave transforms
    $freq1 = $this->_randomFreq();
    $freq2 = $this->_randomFreq();
    $freq3 = $this->_randomFreq();
    $freq4 = $this->_randomFreq();

    $ph1 = $this->_randomPhase();
    $ph2 = $this->_randomPhase();
    $ph3 = $this->_randomPhase();
    $ph4 = $this->_randomPhase();

    $szx = $this->_randomSize();
    $szy = $this->_randomSize();

    for ($x = 0; $x < $w; $x++) {
        for ($y = 0; $y < $h; $y++) {
            $sx = $x + (sin($x*$freq1 + $ph1) + sin($y*$freq3 + $ph3)) * $szx;
            $sy = $y + (sin($x*$freq2 + $ph2) + sin($y*$freq4 + $ph4)) * $szy;

            if ($sx < 0 || $sy < 0 || $sx >= $w - 1 || $sy >= $h - 1) {
                continue;
            } else {
                $color    = (imagecolorat($img, $sx, $sy) >> 16)         & 0xFF;
                $color_x  = (imagecolorat($img, $sx + 1, $sy) >> 16)     & 0xFF;
                $color_y  = (imagecolorat($img, $sx, $sy + 1) >> 16)     & 0xFF;
                $color_xy = (imagecolorat($img, $sx + 1, $sy + 1) >> 16) & 0xFF;
            }
            if ($color == 255 && $color_x == 255 && $color_y == 255 && $color_xy == 255) {
                // ignore background
                continue;
            } elseif ($color == 0 && $color_x == 0 && $color_y == 0 && $color_xy == 0) {
                // transfer inside of the image as-is
                $newcolor = 0;
            } else {
                // do antialiasing for border items
                $frac_x  = $sx-floor($sx);
                $frac_y  = $sy-floor($sy);
                $frac_x1 = 1-$frac_x;
                $frac_y1 = 1-$frac_y;

                $newcolor = $color    * $frac_x1 * $frac_y1
                          + $color_x  * $frac_x  * $frac_y1
                          + $color_y  * $frac_x1 * $frac_y
                          + $color_xy * $frac_x  * $frac_y;
            }
            imagesetpixel($img2, $x, $y, imagecolorallocate($img2, $newcolor, $newcolor, $newcolor));
        }
    }

    // generate noise
    for ($i=0; $i<$this->_dotNoiseLevel; $i++) {
        imagefilledellipse($img2, mt_rand(0,$w), mt_rand(0,$h), 2, 2, $text_color);
    }
    for ($i=0; $i<$this->_lineNoiseLevel; $i++) {
       imageline($img2, mt_rand(0,$w), mt_rand(0,$h), mt_rand(0,$w), mt_rand(0,$h), $text_color);
    }

    imagepng($img2, $img_file);
    imagedestroy($img);
    imagedestroy($img2);
}
```

基本步骤

1. 使用基底图生成缓冲图片
2. 输出验证码文本
2. 根据点和线的噪音级别生成噪音图形
3. 进行波形扭曲变换
4. 再次生成噪声点和线
5. 保存图片到PNG格式的文件

** 验证码管理 **

验证码管理的基础工作在 `Zend_Captcha_Word`中完成，基本实现思路

* 将生成的验证码code存入session中

	> Zend/Captcha/Word.php
	
	```
	public function generate()
    {
        if (!$this->_keepSession) {
            $this->_session = null;
        }
        $id = $this->_generateRandomId();
        $this->_setId($id);
        $word = $this->_generateWord();
        $this->_setWord($word);
        return $id;
    }
	```
	
	```
    protected function _setWord($word)
    {
        $session       = $this->getSession();
        $session->word = $word;
        $this->_word   = $word;
        return $this;
    }
	```
	
* 验证时直接从session中读取并验证

	> Zend/Captcha.Word.php
	
	```
	public function isValid($value, $context = null)
    {
        ...

        $this->_id = $value['id'];
        if ($input !== $this->getWord()) {
            $this->_error(self::BAD_CAPTCHA);
            return false;
        }

        return true;
    }
    
    public function getWord()
    {
        if (empty($this->_word)) {
            $session     = $this->getSession();
            $this->_word = $session->word;
        }
        return $this->_word;
    }
	```
	
* 清除session中的验证码

	通过控制Session数据失效时间和步数来清除Session中的验证码

	> Zend/Captcha.Word.php
	
	```
    public function getSession()
    {
        if (!isset($this->_session) || (null === $this->_session)) {
            $id = $this->getId();
            if (!class_exists($this->_sessionClass)) {
                //require_once 'Zend/Loader.php';
                Zend_Loader::loadClass($this->_sessionClass);
            }
            $this->_session = new $this->_sessionClass('Zend_Form_Captcha_' . $id);
            $this->_session->setExpirationHops(1, null, true);
            $this->_session->setExpirationSeconds($this->getTimeout());
        }
        return $this->_session;
    }
	```
	
** 清理失效验证码图片 **

遍历验证码存储的文件夹，按创建世界删除有效期（最长有效期默认10分钟）之外的图片文件。

参照 
> Zend/Captcha/Image.php#_gc

GC在生成验证码的时候按概率随机触发

> Zend/Captcha/Image.php#generate

```
if (mt_rand(1, $this->getGcFreq()) == 1) {
    $this->_gc();
}
```

除自己生成并管理验证码外，Zend Captcha还支持直接使用`reCaptcha`服务来管理验证码，

## 分页组件的实现
分页是列表展示常见的需求，因而wecenter提供了一个基础的分页工具类 `core_pagination`。

只有提供

* base_url
* total_rows
* per_page

就能生成一个分页导航视图

### 使用

```
TPL::assign('pagination', AWS_APP::pagination()->initialize(array(
				'base_url' => get_js_url('/admin/approval/list/type-' . $_GET['type']),
				'total_rows' => $found_rows,
				'per_page' => $this->per_page
			))->create_links());
```

```
<span class="pull-right mod-page"><?php echo $this->pagination; ?></span>
```

### 实现

基本可以定义参数

> views/default/config/pagination.php

```
$config = array(
	'first_link' => '&lt;&lt;',
	'next_link' => '&gt;',
	'prev_link' => '&lt;',
	'last_link' => '&gt;&gt;',
	'uri_segment' => 3,
	'full_tag_open' => '<div class="page-control"><ul class="pagination pull-right">',
	'full_tag_close' => '</ul></div>',
	'first_tag_open' => '<li>',
	'first_tag_close' => '</li>',
	'last_tag_open' => '<li>',
	'last_tag_close' => '</li>',
	'first_url' => '', // Alternative URL for the First Page.
	'cur_tag_open' => '<li class="active"><a href="javascript:;">',
	'cur_tag_close' => '</a></li>',
	'next_tag_open' => '<li>',
	'next_tag_close' => '</li>',
	'prev_tag_open' => '<li>',
	'prev_tag_close' => '</li>',
	'num_tag_open' => '<li>',
	'num_tag_close' => '</li>',
	'display_pages' => TRUE,
	'anchor_class' => '',
	'num_links' => 3
);
```

具体逻辑参照 

> system/core/pagination.php#create_links

基本思路

* 根据总记录数算出总页数
* 根据当前页和总页数和配置的标签生成分页html代码

## 表单防CSRF(Cross-site request forgery)的实现

* 在生成的表单中插入csrf key隐藏字段

```
<input type="hidden" name="post_hash" value="<?php echo new_post_hash(); ?>" />
```

> system/functions.inc.php

```
function new_post_hash()
{
	if (! AWS_APP::session()->client_info)
	{
		return false;
	}

	return AWS_APP::form()->new_post_hash();
}
```

> system/core/form.php

```
public function __construct()
{
	$this->csrf_key = md5(G_COOKIE_HASH_KEY . $_SERVER['HTTP_USER_AGENT'] . AWS_APP::user()->get_info('uid') . session_id());
}

public function new_post_hash()
{
	return $this->csrf_key;
}
```

可见，假如session有效，所有表单请求csrf_key都是相同的，因而不需要考虑csrf_key的存储和清理问题，是一个很取巧的方法。

* 在表单处理的action里验证csrf key

```
if (!valid_post_hash($_POST['post_hash']))
{
	H::ajax_json_output(AWS_APP::RSM(null, '-1', AWS_APP::lang()->_t('页面停留时间过长,或内容已提交,请刷新页面')));
}

```

> system/functions.inc.php

function valid_post_hash($hash)
{
	return AWS_APP::form()->valid_post_hash($hash);
}

> system/core/form.php

```
public function valid_post_hash($post_hash)
{
	if ($post_hash == $this->csrf_key)
	{
		return TRUE;
	}

	return FALSE;
}
```


## 上传图片并生成缩略图

这部分主要由两个模块合力完成

* core_upload
* core_image

### 上传图片

#### 使用

```
AWS_APP::upload()->initialize(array(
    'allowed_types' => get_setting('allowed_upload_types'),
    'upload_path' => get_setting('upload_dir') . '/' . $item_type . '/' . gmdate('Ymd'),
    'is_image' => FALSE,
    'max_size' => get_setting('upload_size_limit')
));

...

AWS_APP::upload()->do_upload('qqfile');

...

if (AWS_APP::upload()->get_error())
{
    switch (AWS_APP::upload()->get_error())
    {
        default:
        	H::ajax_json_output(AWS_APP::RSM(null, -1, AWS_APP::lang()->_t('错误代码: '.AWS_APP::upload()->get_error())));
        break;

        case 'upload_invalid_filetype':
            H::ajax_json_output(AWS_APP::RSM(null, -1, AWS_APP::lang()->_t('文件类型无效')));
        break;

        case 'upload_invalid_filesize':
           H::ajax_json_output(AWS_APP::RSM(null, -1, AWS_APP::lang()->_t("文件尺寸过大,最大允许尺寸为 ".get_setting('upload_size_limit')." KB")));
        break;
    }
}

if (! $upload_data = AWS_APP::upload()->data())
{
    H::ajax_json_output(AWS_APP::RSM(null, -1, AWS_APP::lang()->_t( '上传失败, 请与管理员联系' )));
}

```

步骤

* 初始化（接受的文件类型、保存路径、是否必须是图片、最大文件尺寸）
* 接受上传文件
* 错误处理

#### 实现
core_upload处理文件上传，涵盖了

#### 文件存储和访问暴露

先会将文件存储到临时文件中，在完成后面的检查后再移动到目标目录

#### 文件类型判定并只允许上传制定类型的文件(mimes)
	
根据上传文件的扩展名判定是否是allowed_types，
如果是图片文件，还会尝试获取图片尺寸（利用获取图片大小会解析图片头部信息的副作用）
另外还是通过函数`finfo_file`、`file`命令或者mime_content_type来分析文件的mimes type

> system/core/upload.php#file_mime_type

```
// We'll need this to validate the MIME info string (e.g. text/plain; charset=us-ascii)
   $regexp = '/^([a-z\-]+\/[a-z0-9\-\.\+]+)(;\s.+)?$/';

```

1. 优先先会尝试使用 ｀finfo_file｀

	```
	/* Fileinfo extension - most reliable method
     *
     * Unfortunately, prior to PHP 5.3 - it's only available as a PECL extension and the
     * more convenient FILEINFO_MIME_TYPE flag doesn't exist.
     */
    if (function_exists('finfo_file'))
    {
        $finfo = finfo_open(FILEINFO_MIME);
        if (is_resource($finfo)) // It is possible that a FALSE value is returned, if there is no magic MIME database file found on the system
        {
            $mime = @finfo_file($finfo, $file['tmp_name']);
            finfo_close($finfo);

            /* According to the comments section of the PHP manual page,
             * it is possible that this function returns an empty string
             * for some files (e.g. if they don't exist in the magic MIME database)
             */
            if (is_string($mime) && preg_match($regexp, $mime, $matches))
            {
                $this->file_type = $matches[1];
                return;
            }
        }
    }
	```
2. 其次是`file --brief --mime`命令

	```
	/* This is an ugly hack, but UNIX-type systems provide a "native" way to detect the file type,
     * which is still more secure than depending on the value of $_FILES[$field]['type'], and as it
     * was reported in issue #750 (https://github.com/EllisLab/CodeIgniter/issues/750) - it's better
     * than mime_content_type() as well, hence the attempts to try calling the command line with
     * three different functions.
     *
     * Notes:
     *  - the DIRECTORY_SEPARATOR comparison ensures that we're not on a Windows system
     *  - many system admins would disable the exec(), shell_exec(), popen() and similar functions
     *    due to security concerns, hence the function_exists() checks
     */
    if (DIRECTORY_SEPARATOR !== '\\')
    {
        $cmd = 'file --brief --mime ' . @escapeshellarg($file['tmp_name']) . ' 2>&1';

        if (function_exists('exec'))
        {
            /* This might look confusing, as $mime is being populated with all of the output when set in the second parameter.
             * However, we only neeed the last line, which is the actual return value of exec(), and as such - it overwrites
             * anything that could already be set for $mime previously. This effectively makes the second parameter a dummy
             * value, which is only put to allow us to get the return status code.
             */
            $mime = @exec($cmd, $mime, $return_status);
            if ($return_status === 0 && is_string($mime) && preg_match($regexp, $mime, $matches))
            {
                $this->file_type = $matches[1];
                return;
            }
        }

        if ( (bool) @ini_get('safe_mode') === FALSE && function_exists('shell_exec'))
        {
            $mime = @shell_exec($cmd);
            if (strlen($mime) > 0)
            {
                $mime = explode("\n", trim($mime));
                if (preg_match($regexp, $mime[(count($mime) - 1)], $matches))
                {
                    $this->file_type = $matches[1];
                    return;
                }
            }
        }

        if (function_exists('popen'))
        {
            $proc = @popen($cmd, 'r');
            if (is_resource($proc))
            {
                $mime = @fread($proc, 512);
                @pclose($proc);
                if ($mime !== FALSE)
                {
                    $mime = explode("\n", trim($mime));
                    if (preg_match($regexp, $mime[(count($mime) - 1)], $matches))
                    {
                        $this->file_type = $matches[1];
                        return;
                    }
                }
            }
        }
    }
	```

3. 然后 `mime_content_type`
	```
	// Fall back to the deprecated mime_content_type(), if available (still better than $_FILES[$field]['type'])
    if (function_exists('mime_content_type'))
    {
        $this->file_type = @mime_content_type($file['tmp_name']);
        if (strlen($this->file_type) > 0) // It's possible that mime_content_type() returns FALSE or an empty string
        {
            return;
        }
    }
	```
	
4. 最后才会考虑用 `getimagesize`
	```
    if (function_exists('getimagesize'))
    {
        $imageinfo = @getimagesize($file['tmp_name']);

        if ($imageinfo['mime'])
        {
            $this->file_type = $imageinfo['mime'];

            return;
        }
    }

    $this->file_type = $file['type'];
	```
	
#### 文件大小检查
#### 图片尺寸检查
#### 文件名和文件内容安全校验（xss clean）

会去掉文件名中的各类非法字符

> system/core/upload.php#clean_file_name

```
    $bad = array(
                   "<!--",
                   "-->",
                   "'",
                   "<",
                   ">",
                   '"',
                   '&',
                   '$',
                   '=',
                   ';',
                   '?',
                   '/',
                   "%20",
                   "%22",
                   "%3c",      // <
                   "%253c",    // <
                   "%3e",      // >
                   "%0e",      // >
                   "%28",      // (
                   "%29",      // )
                   "%2528",    // (
                   "%26",      // &
                   "%24",      // $
                   "%3f",      // ?
                   "%3b",      // ;
                   "%3d"       // =
               );

   $filename = str_replace($bad, '', $filename);
```

** 文件内容的xss clean ** 

> system/core/upload.php#do_xss_clean

```
if (function_exists('memory_get_usage') && memory_get_usage() && ini_get('memory_limit') != '')
{
    $current = ini_get('memory_limit') * 1024 * 1024;

    // There was a bug/behavioural change in PHP 5.2, where numbers over one million get output
    // into scientific notation.  number_format() ensures this number is an integer
    // http://bugs.php.net/bug.php?id=43053

    $new_memory = number_format(ceil(filesize($file) + $current), 0, '.', '');

    @ini_set('memory_limit', $new_memory); // When an integer is used, the value is measured in bytes. - PHP.net
}

// If the file being uploaded is an image, then we should have no problem with XSS attacks (in theory), but
// IE can be fooled into mime-type detecting a malformed image as an html file, thus executing an XSS attack on anyone
// using IE who looks at the image.  It does this by inspecting the first 255 bytes of an image.  To get around this
// CI will itself look at the first 255 bytes of an image to determine its relative safety.  This can save a lot of
// processor power and time if it is actually a clean image, as it will be in nearly all instances _except_ an
// attempted XSS attack.

if (function_exists('getimagesize') && @getimagesize($file) !== FALSE)
{
    if (($file = @fopen($file, 'rb')) === FALSE) // "b" to force binary
    {
        return FALSE; // Couldn't open the file, return FALSE
    }

    $opening_bytes = fread($file, 256);
    fclose($file);

    // These are known to throw IE into mime-type detection chaos
    // <a, <body, <head, <html, <img, <plaintext, <pre, <script, <table, <title
    // title is basically just in SVG, but we filter it anyhow

    if ( ! preg_match('/<(a|body|head|html|img|plaintext|pre|script|table|title)[\s>]/i', $opening_bytes))
    {
        return TRUE; // its an image, no "triggers" detected in the first 256 bytes, we're good
    }
    else
    {
        return FALSE;
    }
}

if (($data = @file_get_contents($file)) === FALSE)
{
    return FALSE;
}

return $data;
```

处理步骤

1. 调整内存`memory_limit`为读入文件保留足够的内存
2. 如果是图片文件，要求文件不能有任何html标签（`a|body|head|html|img|plaintext|pre|script|table|title`）
3. 如果不是图片，只要不为空即可。


### 生成缩略图算法

#### 使用

```
AWS_APP::image()->initialize(array(
    'quality' => 90,
    'source_image' => $upload_data['full_path'],
    'new_image' => $thumb_file[$key],
    'width' => $val['w'],
    'height' => $val['h']
))->resize();
```

core_image负责缩略图的生成

* 生成指定大小的缩略图（resize）
* 可以选择输出到文件或者浏览器

resize的算法支持缩放和裁剪，并支持压缩到一定清晰度

** 缩放 **

* IMAGE_CORE_SC_NOT_KEEP_SCALE 
	
	直接缩放或拉伸，不考虑比例
	
* IMAGE_CORE_SC_BEST_RESIZE_WIDTH

	优先匹配缩放后的目标宽度

* IMAGE_CORE_SC_BEST_RESIZE_HEIGHT

	高度优先匹配
	
** 裁剪 **

* IMAGE_CORE_CM_DEFAULT

	不裁剪
	
* IMAGE_CORE_CM_LEFT_OR_TOP

	对齐左上角，裁剪右下角
	
* IMAGE_CORE_CM_MIDDLE	

	居中对齐，裁剪四周
	
* IMAGE_CORE_CM_RIGHT_OR_BOTTOM

	右下角对齐，裁剪左上角
	
#### 实现

内部支持`GD`或`ImageMagick`来处理图片，支持`jpg/png/gif`三种格式的图片处理。

处理步骤：

1. 根据图片的宽窄比计算目标缩放区域（优先按长边缩放）

	> system/core/image.php#imageProcessImageMagick
	
	```
	if ($this->source_image_w * $this->height > $this->source_image_h * $this->width)
	{
		$match_w = round($this->width * $this->source_image_h / $this->height);
		$match_h = $this->source_image_h;
	}
	else
	{
		$match_h = round($this->height * $this->source_image_w / $this->width);
		$match_w = $this->source_image_w;
	}
	```
	
2. 根据裁剪需求，设定目标区域
	
	> system/core/image.php#imageProcessImageMagick
	
	```
	switch ($this->clipping)
	{
		case IMAGE_CORE_CM_LEFT_OR_TOP:
			$this->source_image_x = 0;
			$this->source_image_y = 0;
		break;

		case IMAGE_CORE_CM_MIDDLE:
			$this->source_image_x = round(($this->source_image_w - $match_w) / 2);
			$this->source_image_y = round(($this->source_image_h - $match_h) / 2);
		break;

		case IMAGE_CORE_CM_RIGHT_OR_BOTTOM:
			$this->source_image_x = $this->source_image_w - $match_w;
			$this->source_image_y = $this->source_image_h - $match_h;
		break;
	}

	$this->source_image_w = $match_w;
	$this->source_image_h = $match_h;
	$this->source_image_x += $this->start_x;
	$this->source_image_y += $this->start_y;
	```
	
3. 根据缩放设置，计算出目标图片的真实尺寸
	
	> system/core/image.php#imageProcessImageMagick
	
	```
	$resize_height = $this->height;
	$resize_width = $this->width;

	if ($this->scale != IMAGE_CORE_SC_NOT_KEEP_SCALE)
	{
		if ($this->scale == IMAGE_CORE_SC_BEST_RESIZE_WIDTH)
		{
			$resize_height = round($this->width * $this->source_image_h / $this->source_image_w);
			$resize_width = $this->width;
		}
		else if ($this->scale == IMAGE_CORE_SC_BEST_RESIZE_HEIGHT)
		{
			$resize_width = round($this->height * $this->source_image_w / $this->source_image_h);
			$resize_height = $this->height;
		}
	}
	```
4. 按目标区域裁剪图片
5. 缩放图片到目标尺寸  
6. 输出图片

具体到图片处理记得，根据库的不同，略有不同

** imagemagick **

```
$im = new Imagick();

$im->readimageblob(file_get_contents($this->source_image));

$im->setCompressionQuality($this->quality);

if ($this->source_image_x OR $this->source_image_y)
{
	$im->cropImage($this->source_image_w, $this->source_image_h, $this->source_image_x, $this->source_image_y);
}

$im->thumbnailImage($resize_width, $resize_height, true);

if ($this->option == IMAGE_CORE_OP_TO_FILE AND $this->new_image)
{
	file_put_contents($this->new_image, $im->getimageblob());
}
else if ($this->option == IMAGE_CORE_OP_OUTPUT)
{
	$output = $im->getimageblob();
			$outputtype = $im->getFormat();

	header("Content-type: $outputtype");
	echo $output;
	die;
}
```

** gd **

通过 `imagecopyresampled`或`imagecopyresized`可以一步作为裁剪和缩放

```
$func_resize($dst_img, $im, $dst_x, $dst_y, $this->source_image_x, $this->source_image_y, $fdst_w, $fdst_h, $this->source_image_w, $this->source_image_h);

```

## 对称加密

### php对称加密介绍

> 参照 [块密码的工作方式](https://zh.wikipedia.org/wiki/%E5%9D%97%E5%AF%86%E7%A0%81%E7%9A%84%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)

基本的思路:

1. 将数据切分成固定大小的数据块(分组)处理

	对确定长度的分组数据处理时（ECB/CBC），如果最后一块的数据不足一块（16字节），需要进行`填充`，比如
	
	> `PKCS#5`
	>
	>
	> * 在最后一块末尾填充5个字节的0x05(填充的字节都是一个相同的字节，该字节的值,就是要填充的字节的个数)
	> * 如果数据刚好一块数据，则附加一整块的0xFF(16字节的填充Ox10)
	>


2. 根据不同的加密模式和算法对数据块加密

wecenter将用户信息（uid、用户名、密码）通过对称加密的方式存储到cookie中。

** 支持各种加密模式 **

除ECB和CBC是固定块长度加密外，其它的（CFB/OFB/CTR）都是可变块长度加密。

* MCRYPTE_MODEL_ECB(Electronic codebook，电子密码本)

	数据块各自加密，数据块直接互不相关
	
	优点:
	1. 简单；
	2. 有利于并行计算；
	3. 误差不会被传送；
	
	缺点:
	1. 不能隐藏明文的模式；
	2. 可能对明文进行主动攻击；

* MCRYPTE_MODEL_CBC(Cipher-block chaining，密码块链)

	上一块的密文块和当前明文块做xor运算后再做加密
	优点：
	1. 不容易主动攻击,安全性好于ECB,适合传输长度长的报文,是SSL、IPSec的标准。
	
	缺点：
	1. 不利于并行计算；
	2. 误差传递；
	3. 需要初始化向量IV

* MCRYPTE_MODEL_CFB（Cipher feedback，密码反馈）

	将前一块加密后输出和当前明文块做xor运算后形成新的密文流入下一块的加密过程
	
	优点：
	1. 隐藏了明文模式;
	2. 分组密码转化为流模式;
	3. 可以及时加密传送小于分组的数据;
	
	缺点:
	1. 不利于并行计算;
	2. 误差传送：一个明文单元损坏影响多个单元;
	3. 唯一的IV;

* MCRYPTE_MODEL_NCFB（N bits Cipher feedback， N位密码反馈模式）

	与CFB过程一样，只是每次仅将前一块加密后输出的高n位和当前明文块高n位xor

* MCRYPTE_MODEL_OFB（Output Feedback, 输出反馈模式）

	将前一块的加密后的输出流入下一块的加密过程，而将明文块和加密后的输出做xor作为密文块	 
* MCRYPTE_MODEL_NOFB(N bits output feedback, N位输出反馈模式)

	与NOFB过程一样，只是每次仅将前一块加密后输出的高n位流入下个过程

* MCRYPTE_MODEL_CTR  

	先将一个自增的算子加密，然后将密文与明文做xor，这样相当于每次秘钥都不一样。
	
* MCRYPTE_MODEL_STREAM

	!!! 没有搞清楚

** PHP通过`mcrypt`加密扩展函数，可以支持各种加密算法 **

> MCRYPT_ciphername

* des
* rc2
* cast-128
* cast-256 
* gost 
* rijndael-128 
* rijndael-192
* rijndael-256
* twofish 
* arcfour 
* loki97 
* saferplus 
* wake   
* serpent 
* xtea 
* blowfish 
* blowfish-compat 
* enigma  
* tripledes

### mcrypt使用

** 加密 **

> system/core/crypte.php#encode

```
public function encode($data, $key = null)
{
    $mcrypt = mcrypt_module_open($this->get_algorithms(), '', MCRYPT_MODE_ECB, '');

    mcrypt_generic_init($mcrypt, $this->get_key($mcrypt, $key), mcrypt_create_iv(mcrypt_enc_get_iv_size($mcrypt), MCRYPT_RAND));

    $result = mcrypt_generic($mcrypt, gzcompress($data));

    mcrypt_generic_deinit($mcrypt);
    mcrypt_module_close($mcrypt);

    return $this->get_algorithms() . '|' . $this->str_to_hex($result);
}
```

步骤：

1. 根据加密算法和加密模式获取加密模块
2. 初始化随机向量
3. 初始化加密器
4. 加密
5. 释放资源
6. 关闭加密模块

** 解密

> system/core/crypte.php#decode

```
public function decode($data, $key = null)
{
   if (strstr($data, '|'))
   {
		$decode_data = explode('|', $data);

		$algorithm = $decode_data[0];

       $data = str_replace($algorithm . '|', '', $data);

       $data = $this->hex_to_str($data);
   }
   else
   {
       $algorithm = $this->get_algorithms();

       $data = base64_decode($data);
   }

   $mcrypt = mcrypt_module_open($algorithm, '', MCRYPT_MODE_ECB, '');

   mcrypt_generic_init($mcrypt, $this->get_key($mcrypt, $key), mcrypt_create_iv(mcrypt_enc_get_iv_size($mcrypt), MCRYPT_RAND));

   $result = trim(mdecrypt_generic($mcrypt, $data));

   mcrypt_generic_deinit($mcrypt);
   mcrypt_module_close($mcrypt);

   return gzuncompress($result);
}
```

步骤：

1. 根据加密算法和加密模式获取模块
2. 初始化随机向量
3. 初始化加密器
4. 解密密
5. 释放资源
6. 关闭模块

## 国际化和多语言

通过 `core_lang`来应对多语言文本的localization

wecenter并未针对用户提供本地化的支持，如果需要提供基于用户的，可以稍加改动，根据浏览器的本地化初始化语种，自动选择本地化资源。

### 使用

配置

> language/en_US.php

```
$language['抱歉, 你的账号已经被禁止登录'] = 'Sorry, your account has been suspended';
```

使用

```
<th><?php _e('文章标题'); ?></th>
```

### 各语种资源配置

提供了一个工具函数，方便使用

> system/functions.inc.php


```
function _e($string, $replace = null)
{
	if (!class_exists('AWS_APP', false))
	{
		echo load_class('core_lang')->translate($string, $replace, TRUE);
	}
	else
	{
		echo AWS_APP::lang()->translate($string, $replace, TRUE);
	}
}
```

将各国语言的配置文件放到 `language`文件夹中
通过系统设置的语言SYSTEM_LANG，来加载不同的文件，实现本地化。

> system/core/lang.php#__consturct

```
$language_file = ROOT_PATH . 'language/' . SYSTEM_LANG . '.php';

if (file_exists($language_file))
{
	require $language_file;
}
```

> system/core/lang.php

```
public function translate($string, $replace = null, $display = false)
{
	$search = '%s';

	if (is_array($replace))
	{
		$search = array();

		for ($i=0; $i<count($replace); $i++)
		{
			$search[] = '%s' . $i;
		};
	}

	if ($translate = $this->lang[trim($string)])
	{
		if (isset($replace))
		{
			$translate = str_replace($search, $replace, $translate);
		}

		if (!$display)
		{
			return $translate;
		}

		echo $translate;
	}
	else
	{
		if (isset($replace))
		{
			$string = str_replace($search, $replace, $string);
		}

		return $string;
	}
}
```

备注

* 构造查找数组
* 如果找不到配置，直接返回key
* 如果需要格式化（替换），执行替换
* 如果需要显示，直接echo到输出流

## 全局异常处理

通过设置全局异常回调，将错误定向到500页面

> system/aws_app.inc.php#init

```
set_exception_handler(array('AWS_APP', 'exception_handle'));

...

public static function exception_handle($exception)
{
	$exception_message = "Application error\n------\nMessage: " . $exception->getMessage() . "\nFile: " . $exception->getFile() . "\nLine: " . $exception->getLine() . "\n------\nBuild: " . G_VERSION . " " . G_VERSION_BUILD . "\nPHP Version: " . PHP_VERSION . "\nURI: " . $_SERVER['REQUEST_URI'] . "\nUser Agent: " . $_SERVER['HTTP_USER_AGENT'] . "\nAccept Language: " . $_SERVER['HTTP_ACCEPT_LANGUAGE'] . "\nIP Address: " . fetch_ip() . "\n------\n" . $exception->getTraceAsString();

	show_error($exception_message, $exception->getMessage());
}
```

> system/functions.inc.php

```
function show_error($exception_message, $error_message = '')
{
	@ob_end_clean();

	if (get_setting('report_diagnostics') == 'Y' AND class_exists('AWS_APP', false))
	{
		AWS_APP::mail()->send('wecenter_report@wecenter.com', '[' . G_VERSION . '][' . G_VERSION_BUILD . '][' . base_url() . ']' . $error_message, nl2br($exception_message), get_setting('site_name'), 'WeCenter');
	}

	if (isset($_SERVER['SERVER_PROTOCOL']) AND strstr($_SERVER['SERVER_PROTOCOL'], '/1.0') !== false)
	{
		header("HTTP/1.0 500 Internal Server Error");
	}
	else
	{
		header("HTTP/1.1 500 Internal Server Error");
	}

	echo _show_error($exception_message);
	exit;
}
```

发生未处理异常时会给wecenter发邮件，不需要这个行为可以关闭掉`report_diagnostics`设置项

## BBCode（Bulletin Board Code）

> 参照：［wikipedia: BBCode］(https://zh.wikipedia.org/wiki/BBCode)

### 使用

BBCode并没有一个共同的标准，但仍然有一些语法因为被广泛采用而成为共通语法：

* [b]粗體[/b]
* [i]斜體[/i]
* [u]底線[/u]
* [url]http://wikipedia.org[/url]
* [url=http://wikipedia.org]Wikipedia[/url]
* [img]http://upload.wikimedia.org/wikipedia/commons/thumb/6/63/Wikipedia-logo.png/72px-Wikipedia-logo.png[/img]
* [quote]引言[/quote]
* [code]Monospace固定字元寬度[/code]
* [size=24]文字[/size]
* [color=red]紅字[/color]
* [:-)]或:smile:

```
$article_info['message'] = FORMAT::parse_attachs(nl2br(FORMAT::parse_bbcode($article_info['message'])));
```

### 实现

参照FORMAT的实现

> system/class/cls_format.inc.php#parse_bbcode

```
public static function parse_bbcode($text)
{
	if (!$text)
	{
		return false;
	}

	return self::parse_links(load_class('Services_BBCode')->parse($text));
}
```

`Services_BBCode`完成了BBCode到html的转换

> system/Services/BBCode.php


2016.07 by haiyang@杭州