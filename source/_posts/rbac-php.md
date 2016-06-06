---
title: rbac分析（php源码）
date: 2016-05-23 22:23:47
tags:
	- 源码
	- php
---
## rbac简介
>基于角色的访问控制（Role-Based Access Control）作为传统访问控制（自主访问，强制访问）的有前景的代替受到广泛的关注。在RBAC中，权限与角色相关联，用户通过成为适当角色的成员而得到这些角色的权限。这就极大地简化了权限的管理。在一个组织中，角色是为了完成各种工作而创造，用户则依据它的责任和资格来被指派相应的角色，用户可以很容易地从一个角色被指派到另一个角色。角色可依新的需求和系统的合并而赋予新的权限，而权限也可根据需要而从某角色中回收。角色与角色的关系可以建立起来以囊括更广泛的客观情况。

<!-- more -->
用一张图来简单地描述一下

![rbac图解](/assets/img/rbac_intro.png)

rbac有3个重要概念：**用户**、**角色**、**权限**。通俗地来说，就是把若干个**权限**分配给某个**角色**，然后在需要时把若干个**角色**分配给指定的**用户**，rbac就是通过这种方式实现访问控制的。

管理员通过分配给一个用户角色来允许该用户可以做某些事情。

**RBAC支持三个著名的安全原则**：
1. 最小权限原则
将其角色配置成其完成任务所需要的最小的权限集
2. 责任分离原则
可以通过调用相互独立互斥的角色来共同完成敏感的任务而体现，比如要求一个计帐员和财务管理员共参与同一过帐
3. 数据抽象
可以通过权限的抽象来体现，如财务操作用借款、存款等抽象权限，而不用操作系统提供的典型的读、写、执行权限


## 为什么使用rbac
普通的ACL在权限越来越多的时候需要维护的权限太多，这造成了ACL的瓶颈。而rbac可以有效地解决这个问题。

## 特点
* 仍然有很多权限存在于系统（问题）
* 人员移动的时候只需要改变人员的角色
* 维护大量的权限仍然是个问题
* 维护分配给每个角色的权限比较容易
* 角色的权限分配需要双重检查确保不会分配错误的权限给任何角色

## 一款开源的rbac库——PHP-RBAC
### 简介
[PHP-RBAC](http://phprbac.net/ "PHP-RBAC")是php的一个简单库，实现了rbac一些基本的功能（不包括用户组），它为开发者提供了NIST Level 2 Standard Role Based Access Control。

下面是PHP-RBAC的一个demo：它实现了角色的分层管理，更贴近实际。
![rbac-demo](http://phprbac.net/img/rbac.png)

我自己简单地试了下库，代码是这样：
``` php
<?php
// turn on all errors
error_reporting(E_ALL);
use PhpRbac\Rbac;
// autoloader
require dirname(__DIR__) . '/autoload.php';

$test = new Test();
$test->myTest();
// myTest();

class Test {
	public function test(){
		$rbac = new Rbac('unit_test');
	}

	public function myTest(){

		$rbac = new Rbac();
		$rbac->reset(true);

		// Create a Permission
		$perm_id = $rbac->Permissions->add('delete_posts', 'Can delete forum posts');
		$perm_id2 = $rbac->Permissions->add('add_posts', 'Can add forum posts');

		// Create a Role
		$role_id = $rbac->Roles->add('forum_moderator', 'User can moderate forums');

		// The following are equivalent statements
		$rbac->assign($role_id, $perm_id);
		$rbac->assign($role_id, $perm_id2);

		$rbac->Users->assign($role_id, 5);

		$res = $rbac->Roles->permissions("2");

		foreach ($res as $perm) {
			print($rbac->Permissions->getDescription($perm).' ');
			print($rbac->Permissions->depth($perm).' ');
			print($rbac->Permissions->getPath($perm).' ');
			echo "</br>";
		}

	}
}
```
结果：
![运行结果](/assets/img/phprbac_result.png)
使用非常容易吧，这是一个轻量的库，只有几百k的大小，所以对于一些对权限管理要求不是特别复杂的（没有用户组、分类等）系统可以考虑使用哦！

### 分析

PHP-RBAC的表设计同许多rbac的软件类似：
![rbac表](/assets/img/rbac_table.png)
![php-rbac表](/assets/img/phprbac_table.png)

PHP-RBAC分层实现：
使用树形结构实现（嵌套集合）：
![树形结构数据库实现](/assets/img/phprbac_tree.png)
![php-rbac例子](/assets/img/phprbac_tree_em.png)

End.
*关于rbac的扩展以后有时间再给大家讲讲。*

参考资料：
1. http://baike.baidu.com/link?url=5FPK3srV0UpUKEJProX7MIJDmX4FFlEp8tQI5VQ8-pnI1xMUv8BY9E4TDeM89astDTGW9Mr0uBWOXpwk_2egr_
2. http://phprbac.net/