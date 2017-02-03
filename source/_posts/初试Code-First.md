---
title: 初试Code First
date: 2017-02-03 13:40:14
tags: c#
---
练习Code First
tools:vs2015 sqlServer2012
## 新建项目
新建-项目-Visual C#-windows-控制台应用程序，命名为：CodeFirstDemo。
## 安装EntityFramework程序包
新建好CodeFirstDemo项目后，我们先来安装EntityFramework，项目-管理NuGet程序包-联机-搜索“EntityFramework”，下载安装。
安装完成后会在项目下自动添加EntityFramework引用：

![](http://7xt8vj.com1.z0.glb.clouddn.com/blog_codefirst01.png)

在Program.cs下添加命名空间 `using System.Data.Entity`;

## 创建模型
为了简单我们直接在Program.cs中创建模型,以后可分离出去
```C#
    public class New
    {
        public int NewId { get; set; }
        public string Title { get; set; }
        public int NewTypeId { get; set; }
        public virtual NewType NewType { get; set; }
    }
    public class NewType
    {
        public int NewTypedId { get; set; }
        public string Name { get; set; }
        public int BlogId { get; set; }
        public virtual List<New> New { get; set; }
    }
```
NewId和NewTypeId在创建的时候会做为表的主键，因为包含有ID关键字，如果不包含的话，需要我们指定主键，要不然创建就会报未找到主键的错误，添加命名空间：
`using System.ComponentModel.DataAnnotations;`
并在属性前添加[Key]标注，指示此字段作为主键：
```
[Key]
public int NewId { get; set; }
```
```
[Key]
public int NewTypedId { get; set; }
```
当然除了Key，System.ComponentModel.DataAnnotations命名空间下还有其他的Attribute，这边就不多说，可以查看msdn有关教程：http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.aspx。
需要注意的另外一点，上面定义模型属性的时候有virtual关键字，表示延迟加载，我的理解是这样：当我访问主实体的时候，启动延迟加载，而不会查询数据库的子实体，只有要访问它的时候才会去数据库查询加载，泛型List表示此实体是一对多的关系。
EF默认是启动延迟加载的，我们如果不需要也可以手动禁止：
`db.Configuration.LazyLoadingEnabled = false;`

## 创建上下文DbContext
DBContext，可以理解为是一个容器，里面有对象与数据表的映射关系以及对象本身.
我对EF中上下文的理解：可以把上下文看成一个数据库控制器，我们可以在其中查询、更改、删除数据，然后通过连接获取或是提交。就像是遥控车的遥控器一样，EF是这个遥控器的核心部件，天线是连接字符串，遥控车是数据库，这个比喻可能不是很恰当，但就是这个意思，大家都懂得。
```C#
    public class NewsContext : DbContext
    {
        public DbSet<New> News { get; set; }
        public DbSet<NewType> NewTypes { get; set; }
    }
```
DbSet表示上下文中给定类型的所有实体的集合或可从数据库中查询的给定类型的所有实体的集合，就像是一个包裹，需要什么东西就往里面装什么东西。
## 配置连接字符串
vs默认可以生成数据库，数据库名是项目命名空间+上下文，有时候我们需要自己定义生成数据库名称，或是生成数据库到指定的服务器，而且有时候数据库迁移了，我们不能再重新生成一遍吧，这时候我们就要自定义数据库连接字符串了：
```xml
  <connectionStrings>
    <add name="NewsContext" providerName="System.Data.SqlClient" connectionString="Data Source=(local);Initial Catalog=CodeFirstDemoDB;User ID=sa;Password=sa" />
  </connectionStrings>
  ```
  　需要注意的是：连接字符串的名称必须要和上下文一致，而且connectionStrings必须放在configuration节点内的最下面
## 创建数据库、读/写数据
在Program.cs的Main方法中填写代码：
  ```C#
              using(var db=new NewsContext())
            {
                Console.Write("输入新闻类型标题: ");
                var name = Console.ReadLine();
                var type_Model = new NewType { Name = name };
                db.NewTypes.Add(type_Model);
                db.SaveChanges();

                Console.WriteLine("查询新闻类型标题: ");
                var search_type = Console.ReadLine();
                var query = from b in db.NewTypes
                            where b.Name == search_type
                            select b;

                Console.WriteLine("查询结果: ");
                foreach (var item in query)
                {
                    Console.WriteLine(item.Name);
                }
                Console.ReadKey();
            }
  ```
  上面代码创建一个上下文对象实例，通过该实例添加一个新闻实体，然后通过输入的值，linq查询结果输出。

  ![](http://7xt8vj.com1.z0.glb.clouddn.com/blog_codefirst02.png)

通过上面的操作，我们并没有创建什么连接字符串和数据库操作，但是打开数据库就可以看到我们通过Code First创建的数据库了。

![](http://7xt8vj.com1.z0.glb.clouddn.com/blog_codefirst03.png)

## CodeFirst迁移

关于Code First 迁移其实就是我们在更改模型的时候，数据库要相应的更改。
1. 例如我们上面创建NewTypes表的时候，没有指定字段的长度，默认是创建字段类型是nvarchar(MAX)，有时候我们觉得字段长度太长，需要修改一下字段长度，不要直接去修改数据库，而是在模型中修改：
```
[MaxLength(50)]
public string Name { get; set; }
```
MaxLength就是上面我们说到DataAnnotations命名空间下的类型，这边我们注意下，我们在生成数据库的时候添加了一条数据，Name字段值是“2017年大吉”，看看我们修改字段长度后，字段值是否发生变化？
2. 在程序包管理器控制台输入`Enable-Migrations`命令来启用迁移，运行完成后在项目中会创建一个Migrations文件夹，下来有两类文件：

![](http://7xt8vj.com1.z0.glb.clouddn.com/blog_codefirst04.png)

- Configuration.cs — 此文件包含“迁移”将用来迁移 BloggingContext 的设置。在本演练中不需要进行任何更改，但是，在此处可以指定种子数据、为其他数据库注册提供程序、更改生成迁移的命名空间等。
- <时间戳>_InitialCreate.cs — 这是第一个迁移，它表示已经应用于数据库的更改。应用更改的目的是将其从空数据库迁移至包含博客和文章表的数据库。尽管我们让 Code First 自动创建这些表，现在我们选择“迁移”（已转化为一次“迁移”）。Code First 还在本地数据库中记录：该“迁移”已经应用。文件名中的时间戳用于排序。
 > 需要注意的是：在我们生成数据库的时候，除了New和NewType表外，还有一个系统生成表__MigrationHistory，从表名上就可以看出是迁移历史记录表。

 3. 在程序包管理器控制台输入`Add-Migration Update-NewType-Name`命令，Add-Migration表示增加一个迁移，后面是迁移名称，这个我们可以随便写，运行后会自动检测模型和数据库发生的变化，在Migrations文件夹下会生成一个“201702030624087_Update-NewType-Name.cs”文件，打开我们可以看到更新内容
```
namespace codeFirstDemo.Migrations
{
    using System;
    using System.Data.Entity.Migrations;
    
    public partial class UpdateNewTypeName : DbMigration
    {
        public override void Up()
        {
          AlterColumn("dbo.NewTypes", "Name", c => c.String(maxLength: 50));
        }
        
        public override void Down()
        {
          AlterColumn("dbo.NewTypes", "Name", c => c.String());
        }
    }
}
```
从上面可以看出就是我们上面修改模型的内容，当然你也可以在这上面直接修改，比如你再加一个更新，可以一起提交到数据库。

4. 在程序包管理器控制台输入“Update-Database”命令，表示将所有的迁移应用到数据库，打开数据库，我们看一下效果：

![](http://7xt8vj.com1.z0.glb.clouddn.com/blog_codefirst05.png)

从上面可以看出，字段Name的类型已经修改为nvarchar(50)，而且字段值并没有发生变化，可以证明，Code First迁移并不是删除数据库再创建。

> Code First迁移除了上面说的字段类型修改还有很多内容，比如添加字段，删除字段，删除表等等，但都是大同小异，可以举一反三。

## 示例Demo下载

[codeFirst](http://pan.baidu.com/s/1o7Iip0A)

> 本文参照[初试Code First（附Demo）](http://www.cnblogs.com/xishuai/p/3632304.html).

> Link:[简书](http://www.jianshu.com/p/e858f9bf1aaa)
  github:[blog](https://wwmin.github.io/2017/02/03/%E5%88%9D%E8%AF%95Code-First/)