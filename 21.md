# 第二十一章 性能优化

全书完整目录请见：[Odoo 14开发者指南（Cookbook）第四版](./README.md)

借助于Odoo框架，你可以开发大型且复杂的应用。任何项目成功的关键是良好的性能。本章中，我们将探讨你需要来优化性能的套路和工具。本章中包含的各节是为提升ORM级别的性能，而非客户端或部署端的性能。

本章中，我们将讲解如下小节：

- 记录集的预提取模式
- 内存内缓存 - ormcache
- 生成图像缩略图
- 访问分组数据
- 创建或写入多条记录
- 通过数据库查询访问记录
- Python代码性能分析

## 记录集的预提取模式

在通过记录集访问数据时，它对数据库进行查询。如有包含多条记录的记录集，提取数据会因多个SQL查询而拖慢系统。本节中，我们讨论如何使用预提取模式来解决这一问题。使用预提取模式，可以减少所需查询的次数，提升性能并让系统变快。

### 如何实现...

看以下代码，它是一种常规的计算方法。在这一方法中，self是多条记录的记录集。在直接对记录集进行迭代时，预提取堪称完美：

```
# 正确的预提取
def compute_method(self):
    for rec in self: 
        print(rec.name)
```

在某些情况下，预提取变得更为复杂，如通过browse方法提取数据的时候。在下例中，我们在for循环中逐一浏览记录。这不会有效使用预提取并且它会比平常执行更多的查询：

```
# 错误的预提取
def some_action(self):
    record_ids = []
    self.env.cr.execute("some query to fetch record id")
    for rec in self.env.cr.fetchall():
        record = self.env['res.partner'].browse(rec[0])
        print(record.name)
```

通过向browse方法传递ID列表，可以创建多条记录的记录集。如果对这个记录集执行操作，预提取会完全正常运行：

```
# 正确的预提取
def some_action(self):
    record_ids = []
    self.env.cr.execute("某些获取记录id的查询")
    record_ids = [rec[0] for rec in self.env.cr.fetchall()]
    recordset = self.env['res.partner'].browse(record_ids)
    for record in recordset:
        print(record.name)
```

通过这种方式，我们既不会损失预提取功能，数据又会在单条SQL查询中获取，

### 运行原理...

在处理多记录集时，预提取有且于减少SQL查询数量。通过一次提取数据来实现。通常预提取在Odoo中自动实现，但在某些情况下会丢失掉这一功能，比如像下例中这样分割记录：

```
recs = [r for r in recordset r.id not in [1,2,4,10]]
```

上面的代码会将记录集分割成多个部分，无法使用到预提取。

使用预提取可极大地提升对象关系映射（ORM）的性能。让我们来探讨预提取底层的运行原理。在通过for循环遍历记录集并在第一次迭代中访问字段值时，预提取就开始施展魔力了。迭代中不只是获取当前记录的数据，预提取会获取所有记录的数据。这背后的逻辑是在for循环中访问某字段，而又很可能会在迭代中获取下一条记录的数据。在for循环的第一次迭代中，预提取会获取所有记录集中的数据并将其保存到缓存中。在for循环的下一次迭代中，数据会从这个缓存中提供，而不是进行新的SQL查询。这会将查询数从O(n)减少为O(1)。

我们假定记录集中有10条记录。在处于第一次循环并访问记录的name字段时，它会获取所有10条记录的数据。不仅仅是针对name字段，它还会获取这10条记录的所有字段。在后续的for循环迭代中，数据会从缓存中进行提供。这会将查询的次数由10次减少到1次。

```
for record in recordset:  # 10条记录的记录集
    record.name  # 它会在第一次循环中获取所有10条记录
    record.email  # email 会从缓存中进行提供
```

注意即使这些字段在for循环体中没有使用，预提取也会获取所有字段的值（*2many字段除外）。这是因为获取额外字段相比对每个字段进行多次查询而言对性能的影响微乎其微。

> 📝**注**：有时预获取的字段会降低性能。这些情况下，你可以通过在上下文中向prefetch_fields上下文传递False来禁用预提取，如下：recordset.with_context(prefetch_fields=False)。

预提取机制使用环境缓存来存储并获取记录值。这表示一旦从数据库中提取了记录，后续对字段的调用都通过环境缓存。可以使用env.cache属性来访问环境缓存。要清除缓存，可使用环境中的invalidate_cache()方法。

### 扩展知识...

如果想要分拆记录集，ORM会使用一个新的预提取上下文生成新的记录集。对这种记录集进行操作仅会对它们各自的记录预提取数据。如是想在这个prefetch后预提取所有的记录，可以通过在with_prefetch() 方法中传递这个预提取记录的所有ID进行实现。下例中，我们将记录集分拆为两部分。这里，我们在两个记录集中传递了一个共同的预提取上下文，因此在从其中某一个获取数据时，ORM会获取另一个的数据并将数据放到缓存中以供未来使用：

```
recordset = ... # 假定记录集中有10条记录
recordset1 = recordset[:5].with_prefetch(recordset._ids)
recordset2 = recordset[5:].with_prefetch(recordset._ids)
```

预提取上下文不仅限于分拆记录集。也可以使用with_prefetch() 方法来在多条记录集之间持有共用预提取上下文。这表示在从一条记录提取数据时，它也会提取所有其它记录集的数据。

## 内存内缓存 - ormcache

Odoo框架提供ormcache装饰器来管理内存中的缓存。本节中，我们将探讨如何对函数管理缓存。

### 如何实现...

这个ORM缓存类可在/odoo/tools/cache.py中进行查看。要在文件中使用它们，需要像下面这样进行导入：

```
from odoo import tools
```

在导入这些类之后，就可以使用ORM缓存装饰器了。Odoo提供了不同类型的内存内缓存装饰器。我们将在下面的各版块中了解其中的各个装饰器。

#### ormcache

这是最简单也最常用的缓存装饰器。需要传递方法输出所依赖的参数名。以下为使用ormcache装饰器的示例方法：

```
@tools.ormcache('mode')
def fetch_mode_data(self, mode):
    # 一些运算
    return result
```

初次调用这个方法时，它会进行执行并返回结果，ormcache会根据mode参数的值来存储这一结果 。再次使用相同的mode值调用该方法时，就会不再执行实际方法而从缓存中直接获取结果。

有时，方法的结果依赖于环境属性。在这类情况下，可以声明方法如下：

```
@tools.ormcache('self.env.uid', 'mode')
def fetch_data(self, mode):
    # 一些运算
    return result
```

本例中给出的方法会根据环境用户及mode参数的值存储缓存。

#### ormcache_context

这类缓存与ormcache的非常相似，不同在于它依赖于参数加上下文中的值。在这种缓存装饰器中，需要传递参数名和一个上下文键列表。例如，如果方法的输出依赖于上下文中的lang和website_id键，可以使用ormcache_context：

```
@tools.ormcache_context('mode', keys=('website_id','lang'))
def fetch_data(self, mode):
    # 一些运算
    return result
```

上例中的缓存会依赖于参数mode以及上下文中的值。

#### ormcache_multi

有些方法对多条记录或ID执行操作。如果希望对这类方法添加缓存的话，可以使用ormcache_multi装饰器。需要传递multi参数，并且在方法调用过程中，ORM会通过迭代这一参数生成缓存键。在这个方法中，需要以字典格式返回结果，其中multi参数的元素作为键。看一下如下示例：

```
@tools.ormcache_multi('mode', multi='ids')
def fetch_data(self, mode, ids):
    result = {}
    for i in ids:
        data = ... # 根据ids所做的一些运算
        result[i] = data
    return result
```

假定我们使用 [1,2,3]作为 IDs 来调用以上方法。该方法返回结果的格式为{1:... , 2:..., 3:... }。ORM会根据这些键来缓存结果。如果你以 [1,2,3,4,5] 作为 IDs 来进行另一次调用，方法会在ID参数中接收到[4, 5]，因此该方法会针对 4,5 这两个ID执行操作，并且剩下的结果会由缓存提供。

### 运行原理...

ORM缓存以字典格式（缓存查找）在缓存中保存。这个缓存的键将根据所装饰方法的签名来生成，值即为结果。简单地说，在使用x, y 参数调用该方法时且该方法的结果为x+y，缓存查找即为{(x, y): x+y}。这表示下一次使用相同参数调用这个方法时，结果会通过这个缓存直接进行提供。这节约了运算时间并且让响应更为快速。

ORM缓存是一种内存内的缓存，因此它在RAM中进行存储并占用内存空间。不要使用ormcache来保存大型数据，如图片或文件。

> **⚠️警告：**使用这一装饰器的方法永远不应返回一个记录集。如果这么做，它们会因底层的记录集游标关闭而产生psycopg2.OperationalError。

应该对纯函数使用ORM缓存。纯函数是一种对相同参数一直返回相同结果的方法。这类方法的输出仅依赖于参数，因此它们会返回相同的结果。如果不是这种情况，就需要在执行让缓存状态无效的操作时手动清除缓存。调用clear_caches()方法来清除缓存：

```
self.env[model_name].clear_caches()
```

一旦清除完缓存，下次方法调用会将结果存储到缓存中，随后所有使用相同参数的方法调用都会从缓存中获取。

### 扩展知识...

ORM缓存是一种最近最少使用（LRU）缓存（或称缓存淘汰算法），这表示如果缓存中的一个键不经常使用，就会被删除。如果使用ORM缓存不当，可能会弊大于得。举个例子，如果传入方法的参数总是不同，那么每次Odoo都会先查询缓存然后再调用方法进行运算。如果想要了解缓存是如何执行的，可以向Odoo进程传递SIGUSR1信号：

```
kill -SIGUSR1 2697
```

此处的2697是进程ID。在执行这条命令后，你将可以在日志中查看ORM缓存的状态：

```
> 2020-11-08 09:22:49,350 496 INFO book-db-14 odoo.tools.cache: 1 entries, 31 hit, 1 miss, 0 err, 96.9% ratio, for ir.actions.act_window._existing
> 2020-11-08 09:22:49,350 496 INFO book-db-14 odoo.tools.cache: 1 entries, 1 hit, 1 miss, 0 err, 50.0% ratio, for ir.actions.actions.get_bindings
> 2020-11-08 09:22:49,350 496 INFO book-db-14 odoo.tools.cache: 4 entries, 1 hit, 9 miss, 0 err, 10.0% ratio, for ir.config_parameter._get_param
```

其中的百分比是缓存命中的比例。即在缓存中成功查找到结果的比率。如果命中比率过低，应当在方法中去除掉ORM缓存。

## 生成图像缩略图

超大图片对于网站是一种负担。它们增加了网页的大小并让最终让网站加载速度变慢。这会导致很差的SEO排名和用户丢失。本节中，我们将探讨如何创建不同尺寸的图像，通过使用合适的图片，可以减小网页的大小并改善页面加载时长。

### 如何实现...

需要在模型中继承 image.mixin。下面演示如何在模型中添加 image.mixin：

```
class LibraryBook(models.Model):
    _name = 'library.book'
    _description = 'Library Book'
    _inherit = ['image.mixin']
```

该mixin会自动在图书模型中添加5个新字段来存储不同大小的图片。参见*运行原理...*一节学习这5个字段。

### 运行原理...

image.mixin实例会自动对模型添加5个新的二进制字段。每个字段存储不同分辨率的图片。以下是字段及对应的分辨率：

- **image_1920**: 1,920x1,920
- **image_1024**: 1,024x1,024
- **image_512**: 512x1,512
- **image_256**: 256x256
- **image_128**: 128x128

所有这些字段中，只有image_1920是可编辑的。其它图片字段为只读，在修改image_1920字段时会自动更新。因此，在模型的后台表单视图中，我们使用image_1920字段来让用户上传图片。但这样我们也在表单视图中加载了image_1920大图。不过有一种在表单视图中使用image_1920图但显示小图的方式来改善性能。例如，我们可以使用image_1920但显示image_128字段。使用如下语法实现：

```
<field name="image_1920" widget="image" options="{'preview_image': 'image_128'}" />
```

对字段进行图片保存时，Odoo会自动缩放图片存储到对应的字段中。在表单视图中则会显示转化后的image_128，因为对preview_image使用了该字段。

> 📝**注**：image.mixin模型是一个抽象模型，因此基数据表不在数据库中。需要在模型中继承它来进行使用。

通过image.mixin，我们可以存储的最大分辨率是1,920x1,920。如果存储的图片分辨率大于1,920x1,920，Odoo会将其缩放到1,920x1,920。这么做时，Odoo会保留图片的比例，以免出现拉伸。如果上传的图片分辨率是2,400x1,600，image_1920字段中的分辨率是1,920x1,280。

### 扩展知识...

可通过image.mixin获取特定分辨率的图片，但如果想要使用其它分辨率的图片怎么办呢？可以使用一个二进制封装字段的图片，如下例所示：

```
image_1500 = fields.Image("Image 1500", max_width=1500, max_height=1500)
```

这会新建一个image_1500字段，存储的图片会调整为1,500x1,500的分辨率。注意，这不属于image.mixin。它只是将图片缩放到1,500x1,500，你需要自己将该字段添加到表单视图中，编辑它不会对image.mixin中的其它图片字段作出修改。如果想要将其与已有的image.mixin字段进行关联，在字段定义中要添加**related="image_1920"**的属性。

## 访问分组数据

在想要进行数据统计时，经常需要对数据进行分组，如月销售报表或显示每个客户销售状况的报表 。搜索记录来手动分组会非常耗时。本节中，我们将探讨如何使用 read_group()方法来访问分组数据。

### 如何实现...

执行如下步骤：

> 📝注：read_group()方法广泛用于数据统计和智能统计按钮。

1. 假设你想要在partner表单中显示销售订单的数量。可通过搜索某客户的销售订单然后对长度计数来实现：

   ```
   # in res.partner model
   so_count = fields.Integer(compute='_compute_so_count', string='Sale order count')
   
   def _compute_so_count(self):
       sale_orders = self.env['sale.order'].search(domain=[('partner_id', 'in', self.ids)])
       for partner in self:
           partner.so_count = len(sale_orders.filtered(lambda so: so.partner_id.id == partner.id))
   ```

   上例可以运行，但不够优化。当我们在列表视图中显示so_count字段时，它会获取列表并过滤中的所有人的销售订单。对于这么少量的数据， read_group()方法并不会带来质的变化，但在数据量上升时这就是一个问题了。我们可以使用read_group方法来解决这一问题。

2. 下例中我们将做和前例同样的事情，但即使是对大型数据集它也仅消耗一次SQL查询：

   ```
   # in res.partner model
   so_count = fields.Integer(compute='_compute_so_count', string='Sale order count')
   
   def _compute_so_count(self):
       sale_data = self.env['sale.order'].read_group(
           domain=[('partner_id', 'in', self.ids)],
           fields=['partner_id'], groupby=['partner_id'])
       mapped_data = dict([(m['partner_id'][0], m['partner_id_count']) for m in sale_data])
       for partner in self:
           partner.so_count = mapped_data[partner.id]
   ```

   以上代码段已进行了优化，因其通过SQL的GROUP BY功能直接获取到了销售订单的数量。

**译者注：**partner这个词实在没想到好的翻译，它包含三个部分：供应商(Vendor)、客户(Customer)的雇员(Employee)，既可以是公司，也可以是个人，我能想到最接近的词是主体。

### 运行原理...

read_group()方法在内部使用SQL的GROUP BY功能。这会让read_group方法即使处理大数据集也更为快速。Odoo内部的网页客户端在图表及分组列表视图中使用这一方法。可以通过使用不同的参数来调整read_group方法的行为。

让我们来讨论下read_group方法的声明：

```
def read_group(self, domain, fields, groupby, offset=0, limit=None, orderby=False, lazy=True):
```

针对read_group方法的可用参数如下：

- domain：domain用于过滤记录。用作read_group方法的搜索条件。

- fields：这是一个使用分组获取的字段列表。注意这里讨论的字段应在groupby参数中，除非你使用了聚合函数。从Odoo 12开始，read_group方法支持SQL聚合函数。假设我们想要获取每个客户的平均订单量。这时，可以使用read_group如下：

  ```
  self.env['sale.order'].read_group([], ['partner_id', 'amount_total:avg'], ['partner_id'])
  ```

  如果想要用不同的聚合函数再次访问相同的字段，语法会有一些不同。需要以alias:agg(field_name)的形式传递字段。下例会给出每个客户的订单总量和平均数：

  ```
  self.env['sale.order'].read_group([], ['partner_id', 'total:sum(amount_total)', 'avg_total:avg(amount_total)'], ['partner_id'])
  ```

- groupby：这个参数是对记录分组的字段列表。让我们可以根据多个字段来分组记录。要进行这一实现，需要传递一个字段列表。例如，如果想要通过客户和订单状态来对销售订单分组，可以在参数中传递['partner_id ', 'state']。

- offset：这个参数用于分页。如果想要跳过一些记录，可以使用该参数。

- limit：这个参数用于分页，它表示所要获取记录的最大数量。

- lazy：这个参数接收布尔值。默认值为True。如果该参数为True，结果会仅通过groupby参数中第一个字段来进行分组。可在结果中的__context和__domain键中获取到剩余的groupby参数及作用域。如果将该参数值设为False，它会用groupby参数中的所有字段对数据进行分组。

### 扩展知识...

对日期字段分组会比较复杂，因为可以根据日、周、季度、月或年来对记录进行分组。可以通过在groupby参数的 : 后传递groupby_function来修改日期的分组行为。如果想要对销售订单的月汇总进行分组，可以使用下面的read_group方法：

```
self.env['sale.order'].read_group([], ['total:sum(amount_total)'], ['order_date:month'])
```

对日期分组的可用选项有day, week, month, quarter和year。

### 其它内容

如果想要学习PostgreSQL聚合函数的更多知识，请参见文档：https://www.postgresql.org/docs/current/functions-aggregate.html。

## 创建或写入多条记录

如果你是Odoo开发的新手，可能会执行多条查询来写入或创建多条记录。本节中，我们将了解如何批量创建和写入记录。

### 如何实现...

创建和写入多条记录底层实现方式不同。我们来分别查看。

#### 创建多条记录

Odoo支持指创建记录。如果创建单条记录，只需将字段值加到字典中进行传递即可。而批量创建记录也只需要传递这些词典的列表来代替原来的单个字典。下例中使用单个create调用创建了三条图书记录：

```
vals = [{
    'name': "Book1",
    'date_release': '2018/12/12',
}, {
    'name': "Book2",
    'date_release': '2018/12/12',
}, {
    'name': "Book3",
    'date_release': '2018/12/12',
}]

self.env['library.book'].create(vals)
```

这段代码会创建3本新书的记录。

#### 写入多条记录

如果使用过多个版本的Odoo，应该会知道**write**底层的运行。在版本13中，Odoo对**write**的处理发生了变化 。它使用延迟更新的方式，也就是说并不会即时将数据写入数据库。仅在必要时或在调用**flush()** 时Odoo才会将数据写入数据库，

以下是两个**write**方法的示例：

```
# Example 1
data = {...}
for record in recordset:
    record.write(data)

# Example 2
data = {...}
recordset.write(data)
```

如使用Odoo v13或以上版本，在性能方面就不会有任何问题。但如果使用较早的版本，第二个示例会比第一个更为快速，因为第一条示例会在每次迭代时执行一次SQL查询。

### 运行原理...

要批量创建多条记录，需要以列表的形式传递值字典来新建记录。这会自动管理批量创建的记录。在批量创建记录时，内部会为每条记录插入一个查询。这表示批量的记录创建不是由单条查询实现的。但这并不意味着批量创建不会提升性能。性能的提升是通过批量计算计算字段来实现的。

对于write方法则不同。大部分由框架自动处理。例如，如果在所有记录中写入相同数据，只会使用一条**UPDATE**语句来更新数据库。框架甚至会对相同事务中反复更新同一条记录进行处理，如下：

```
recordset.name= 'Admin'
recordset.email= 'admin@example.com'
recordset.name= 'Administrator'
recordset.email= 'admin-2@example.com'
```

以上代码段中，仅会使用最终值**name=Administrator**和**email=admin-2@example.com**对**write**执行一次查询。这不会影响到性能，因为所赋值存储在缓存中，稍后在同一条语句中进行写入。

如果在中间使用了**flush()** 方法就不太一样了，如下例所示：

```
recordset.name= 'Admin'
recordset.email= 'admin@example.com'
recordset.flush()
recordset.name= 'Administrator'
recordset.email= 'admin-2@example.com'
```

**flush()** 方法会将缓存中的值更新到数据库中。因此上例中会执行两条**UPDATE**语句，一条为flush之前的数据，一条为flush之后的数据。

### 扩展知识...

延迟更新Odoo 13之后才有，如果使用更早的版本，写入单个值会立即执行**UPDATE**语句。看如下示例来了解在更老的Odoo版本中如何正确使用**write**操作：

```
# 错误的用法
recordset.name= 'Admin'
recordset.email= 'admin@example.com'

# 正确的用法
recordset.write({'name': 'Admin', 'email'= 'admin@example.com'})
```

第一个示例中，会执行两条**UPDATE**语句，而第二个示例中仅会执行一条**UPDATE**语句。

## 通过数据库查询访问记录

Odoo ORM的方法有限，并且有时通过ORM来获取某些数据会非常困难。在这类情况下，可以按想要的格式获取数据并需要对数据执行运算来获取特定的结果。因此，速度会变慢。要处理这些特殊情况，可以在数据库中执行SQL查询。本节中我们将探讨如何在Odoo中执行SQL查询。

### 如何实现...

可以通过self._cr.execute方法来执行数据库查询：

1. 添加如下代码：

   ```
   self.flush()
   self._cr.execute("SELECT id, name, date_release FROM library_book WHERE name ilike %s", ('%odoo%',))
   data = self._cr.fetchall()
   print(data)
   
   
   Output:
   [(7, 'Odoo basics', datetime.date(2018, 2, 15)), (8, 'Odoo 11 Development Cookbook', datetime.date(2018, 2, 15)), (1, 'Odoo 12 Development Cookbook', datetime.date(2019, 2, 13))]
   ```

2. 查询的结果格式为元组列表。元组中的数据和查询中字段顺序相同。如果想要以字典格式获取数据，可以使用dictfetchall()方法。参见下例：

   ```
   self.flush()
   self._cr.execute("SELECT id, name, date_release FROM library_book WHERE name ilike %s", ('%odoo%',))
   data = self._cr.dictfetchall()
   print(data)
   
   
   Output:
   [{'id': 7, 'name': 'Odoo basics', 'date_release': datetime.date(2018, 2, 15)}, {'id': 8, 'name': 'Odoo 11 Development Cookbook', 'date_release': datetime.date(2018, 2, 15)}, {'id': 1, 'name': 'Odoo 12 Development Cookbook', 'date_release': datetime.date(2019, 2, 13)}]
   ```

如果希望仅获取单条记录，可以使用fetchone() 和dictfetchone()方法。这些方法与fetchall()和dictfetchall()相似，但仅会返回单条记录，如果想要获取多条记录则需要对它们进行多次调用。

### 运行原理...

通过记录集有两种访问数据库游标的方式：一种是通过记录集本身，例如self._cr，另一种是通过环境，例如self.env.cr。游标用于执行数据库查询。在前例中，我们看到了如何通过原生查询语句获取数据。表名是将模型名中的 . 替换为 _ 之后的名称，因此library.book模型变成了library_book。

如果留意的话，我们在执行语句之前使用了**self.flush()**。原因是Odoo大量地使用了缓存，数据库中值可能还不正确。self.flush()会将所有的延迟更新推送给数据库，并且执行所有的依赖计算，这样就会从数据库中获取正确的值。flush()方法还支持一些参数，协助控制将哪些数据刷入数据库。参数如下：

- **fname** 参数需要一个希望刷入数据库的字段列表。
- **records** 参数需要一个记录集，在仅希望刷入指定记录时使用。

如果执行了INSERT或UPDATE语句，需要在执行语句之后执行 **flush()**，因为ORM可能还不知晓所做的修改，可能存在缓存记录。

在执行原生查询前需要考虑一些事项。仅在没有其它选择时使用原生查询。通过执行原生查询，跳过了ORM层。因此也跳过了安全规则以及丧失了ORM的性能优势。有时错误的语句会带来SQL注入的风险。考虑以下示例，其中查询会允许攻击者执行SQL注入：

```
# very bad, SQL injection possible
self.env.cr.execute('SELECT id, name FROM library_book WHERE name ilike + search_keyword + ';')
# good
self.env.cr.execute('SELECT id, name FROM library_book WHERE name ilike %s ';', (search_keyword,))
```

也不要使用字符串格式函数，这会允许攻击者执行SQL注入。使用SQL查询会让其他开发者更难阅读和理解你的代码，因此应尽可能地避免使用。

> **📝小贴士：**有些Odoo开发者认为执行SQL查询会让操作更快速，因为它绕过了ORM层。但并非完全如此，取决于具体的情况。在某些操作中，执行ORM要比原生查询要更好且更快，因为数据从记录集缓存中进行提供。

### 扩展知识...

一个事务中的操作仅在事务结束时才执行提交。如果在ORM中发生错误，事务会进行回滚。如果你使用INSERT或UPDATE查询并且想让其持久化，可以使用self._cr.commit()来提交这个修改。

> 📝注：注意使用commit() 可能带来风险，因为它将记录置于不连续的状态。ORM中的错误会导致不完整的回滚，因此仅在非常清楚的情况下使用commit()。

如使用了**commit()**方法，则无需再使用**flush()**方法。**commit()** 方法内部将环境刷入数据库。

## Python代码性能分析

有时，会无法定位到问题的原因。尤其是对于性能问题更是如此。Odoo提供了一些内置的性能测试工具来帮助我们发现问题的真实原因。

### 如何实现...

执行如下步骤：

1. Odoo的性能分析工具在odoo/tools/profiler.py中。要在代码中使用这个工具，需要在文件中进行导入：

   ```
   from odoo.tools.profiler import profile
   ```

2. 在导入之后，可以对方法使用profile装饰器。要对指定方法进行性能测试，需要对其添加profile装饰器。参见下例。我们对make_available方法添加了profile装饰器：

   ```
   @profile
   def make_available(self):
       if self.state != 'lost':
           self.write({'state': 'available'})
       return True
   ```

3. 因此，在调用这个方法时，会在日志中打印出完整的统计数据：

   ```
   calls queries ms
   library.book ------------------------
   /Users/pga/odoo/test/my_library/models/library_book.py, 24
   1 0 0.01  @profile
             def make_available(self):
   1 3 12.81   if self.state != 'lost':
   1 7 20.55     self.write({'state': 'available'})
   1 0 0.01    return True
   Total:
   1 10 33.39
   ```

### 运行原理...

在对方法添加profile装饰器之后，调用该方法时Odoo会在日志中打印完整的统计数据，如前例所示。它会以三列打印统计数据。第1列包含调用的次数或某行执行的次数。（这个数据在某行代码位于for循环内或方法是递归的时会逐渐增加）。第2列表示给定行执行查询的次数。最后一列是给定行所花费的毫秒数。注意该列中显示的时间为相对值，在关闭掉性能调试工具后它会更快些。

profiler装饰器接收一些可选参数，这帮助我们获取方法的具体统计数据。以下是profile装饰器的定义：

```
def profile(method=None, whitelist=None, blacklist=(None,), files=None, minimum_time=0, minimum_queries=0):
```

下面是profile()方法所支持的参数列表：

- whitelist：该参数接收一个在日志中显示的模型名称列表。
- files：该参数接收一个显示的文件名列表。
- blacklist：该参数接收一个你不希望在日志中显示的模型名称列表。
- minimum_time：接收一个整型值（毫秒）。在总耗时低于给定值时会隐藏日志。
- minimum_queries：接收一个查询数量的整型值。在总查询数小于给定值时会隐藏日志。

### 扩展知识...

Odoo中另一种类型的性能工具是为执行的方法生成图表。这个性能工具在misc包中，因此需要从那里进行导入。它会生成一个可进一步生成图表文件的统计数据文件。要使用这个性能工具，需要传递文件路径来作为参数。在调用该函数时，它会在给定位置生成一个文件。参见下例，它会在桌面生成一个make_available.prof文件：

```
from odoo.tools.misc import profile

...


@profile('/Users/parth/Desktop/make_available.profile')
def make_available(self):
    if self.state != 'lost':
        self.write({'state': 'available'})
        self.env['res.partner'].create({'name': 'test', 'email': 'test@ada.asd'})
        return True
```

在调用make_available方法时，它会在桌面生成一个文件。要将这个数据转化为图表数据，需要安装gprof2dot工具，然后执行给定的命令来生成图表：

```
gprof2dot -f pstats -o /Users/parth/Desktop/prof.xdot /Users/parth/Desktop/make_available.prof
```

这条命令会在桌面上生成prof.xdot 文件。然后，可以使用下面的命令来通过xdot显示图表：

```
xdot /Users/parth/Desktop/prof.xdot
```

以上的xdot命令会生成下图中的图表：

![图21.1 – 查看执行时间的图表](https://i.cdnl.ink/homepage/wp-content/uploads/2021/04/2021042713491464.jpg)

图21.1 – 查看执行时间的图表

这里你可以放大、查看调用栈，并查看方法执行时间的详情。

