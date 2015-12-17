# PEDT Specifications

Author: aimingoo(aimingoo@wandoujia.com)

Date: 2015.11

Version: Series 1 - 1.1.0

Language: Chinese

# 概要

并行的可交换分布式任务（PEDT, Parallel Exchangeable Distribution Task）是以可计算集群为对象的、以实时处理为目标的并行任务规范。该规范包括对任务数据、任务处理和任务调用接口三个部分的定义。

PEDT旨在为可计算集群提供一种轻量、高效和跨平台的可靠任务处理机制。

### Table of Contents

* [PEDT Specifications](#pedt-specifications)
* [概要](#概要)
* [规范](#规范)
  * [PEDT define specification](#pedt-define-specification)
    * [基础定义](#基础定义)
      * [任务定义（taskDef）](#任务定义taskdef)
      * [任务（task/distribution task）](#任务taskdistribution-task)
      * [任务标识（taskId）](#任务标识taskid)
      * [字符串（string）](#字符串string)
      * [方法(method)](#方法method)
        * [任务分发范围(distribution scope)](#任务分发范围distribution-scope)
        * [任务参数（arguments）](#任务参数arguments)
    * [规范](#规范-1)
      * [PEDT define specification 0.9](#pedt-define-specification-09)
      * [PEDT define specification 1.0](#pedt-define-specification-10)
      * [PEDT define specification 1.1](#pedt-define-specification-11)
    * [keywords](#keywords)
  * [PEDT process specification](#pedt-process-specification)
    * [基础定义](#基础定义-1)
      * [PEDT任务的四个处理阶段](#pedt任务的四个处理阶段)
      * [PEDT任务的三个特定处理方法（process method）](#pedt任务的三个特定处理方法process-method)
      * [PEDT任务的两个特定分发方法（distribution method）](#pedt任务的两个特定分发方法distribution-method)
      * [对处理系统的基本需求](#对处理系统的基本需求)
    * [规范](#规范-2)
      * [PEDT process specification 1.0](#pedt-process-specification-10)
  * [PEDT interface specification](#pedt-interface-specification)
    * [本地接口](#本地接口)
      * [混入对象：mix](#混入对象mix)
      * [HTTP请求：distributed_request](#http请求distributed_request)
      * [方法：taskDef.distributed](#方法taskdefdistributed)
      * [方法：taskDef.promised](#方法taskdefpromised)
      * [方法：taskDef.rejected](#方法taskdefrejected)
      * [解析范围](#解析范围)
      * [下载任务](#下载任务)
      * [执行任务](#执行任务)
      * [方法：task.run](#方法taskrun)
      * [方法：task.map](#方法taskmap)
    * [远程接口](#远程接口)
      * [任务注册/register_task](#任务注册register_task)
      * [任务下载/download_task](#任务下载download_task)
      * [任务执行/execute_task](#任务执行execute_task)
      * [资源query](#资源query)
    * [其它](#其它)
      * [资源订阅/subscribe](#资源订阅subscribe)
      * [资源请求/require](#资源请求require)
      * [方法task.reduce](#方法taskreduce)
      * [方法task.daemon](#方法taskdaemon)
* [实现](#实现)
* [Q&amp;A](#qa)

# 规范

本规范集(specifications)中，任务数据定义(taskDef)以1.0版本为基准，同时发布了taskDef 0.9和1.1共三个标准版本，而对应的任务处理规范(process specification)和任务调用接口规范(interface specification)都只存在1.0版。

taskDef的不同版本用于应对于不同的分布式任务场景，而它们的逻辑处理与外部接口都是一致的。

## PEDT define specification

本规范用于描述一个PEDT任务的可传输数据部分的格式，我们称之为taskDef。

taskDef基于JSON数据定义，0.9版采用受限制的(limited)的JSON格式，而1.0与1.1版本采用完整支持的JSON格式。

### 基础定义

#### 任务定义（taskDef）

一个taskDef总是一个JSON对象(object)，它拥有0至任意多个成员(members/fields)；当它至少有一个成员是分布式任务(distribution task)或处理方法(process method)时，它是分布式的(distributed)，否则它是静态的(static)。

``` text
# sample - static taskDef
{}

# sample - distributed taskDef
{ "x": { "run": "task:570b41ba61ade63987d318b0c08e4fa4" } }
```

一个分布式任务定义(distributed taskDef)也可以处理taskDef自身，后者是将静态任务定义(static taskDef)作为数据对象（而非可执行的任务对象）来处理，例如分发或作为数据规格(data schema)。

在传输中，taskDef是一个缺省使用utf-8编码的标准字符串，并总是可以按JSON规范解析(parse)。

#### 任务（task/distribution task）

一个task总是一个JSON对象(object)，它拥有至少一个成员名（"map"/"run"），用来表示分发方法(distribution method)，以及可能的成员属性，包括scope和arguments等。

如果一个task的map/run成员是任务标识（taskId），那么它是一个需要从远程下载的已发布任务定义(published taskDef)；map成员只接受taskId。

如果一个task的run成员是字符串（string），那么它是被编码在taskDef中可执行的普通本地任务(local task)；如果它是对象(object)，那么它将作为一个本地任务定义(local/unpublished taskDef)来处理。

#### 任务标识（taskId）

taskDef一旦被定义将不可以被变更，因此它具有一个唯一的ID。在PEDT规范中将基于taskId来交换任务，而不是直接交换任务的文本。

PEDT Specifications series 1采用任务文本的MD5值作为taskId，格式为：

``` text
<prefix><md5>
	prefix  : 前缀"task:"
	md5     : taskDef文本的md5值，小写
```

taskId在taskDef中被记为一个字符串值。

#### 字符串（string）

在taskDef中存在两种字符串：一般的(normal)和编码的(encoded)，都是标准的JSON字符串值。编码字符串(encoded string)采用前缀来表达编码内容的含义，需要由处理系统在得到taskDef时进行解码(decode)。

编码字符串采用三段前缀(three parts prefix)格式为：

``` text
<type[:subType]>[:encodeType]:
    type       : string
    subType    : <type define>
	encodeType : string
```

目前PEDT Specifications series 1只支持"data"、"script"两个顶级type前缀。在顶级type前缀为data时，subType缺省为"string"；而顶级type前缀script没有缺省的subType值。

> NOTE: 在规范0.9中，"task:"也被作为顶级type前缀来理解，但在规范1.0及以上版本中，它是被转换成标准的task对象来处理的。
> 
> NOTE: 三段前缀(three parts prefix)是中位缺省的嵌套格式，它的有效分段数并不确定（大于等于1）。当它的encodeType有效时，subType才可能被有效置值（或使用缺省值）。并且，在后续的内容区(body/context)不能出现":"字符。

目前PEDT Specifications series 1支持base64、utf8两种编码类型(encodeType)，缺省为"utf8"。

因此如下两个定义是指一般字符串"Hello World!"：

``` text
"data:Hello World!"
"data:string:utf8:Hello World!"
	NOTE1: "data" is enocded by utf8, and
	NOTE2: "string" is new type as subType
```

而如下三个定义都是指编码为base64的字符串"Hello World!"：

``` text
"data:base64:SGVsbG8gV29ybGQhCg=="
"data:string:base64:SGVsbG8gV29ybGQhCg=="
"data:string:utf8:base64:SGVsbG8gV29ybGQhCg=="
```

> NOTE: "string"不是顶级type前缀，而是"data"类型的缺省subType。

下面的示例声明是等义的：

``` text
"script:lua:base64:cHJpbnQoImhpIikK"
"script:javascript:base64:Y29uc29sZS5sb2coImhpIikK"
"script:lua:utf8:print(\"hi\")"
"script:javascript:utf8:console.log(\"hi\")"
```

> NOTE: "script"前缀没有缺省subType，因此必须显式声明。

#### 方法(method)

方法包括处理方法（process method）与分发方法（distribution method）。处理方法作用于taskDef，而分发方法作用于task。

在taskDef文本中，方法总是表达为编码字符串(encoded string for taskDef），或任务标识(taskId for task)。对于编码字符串，它在解码后是可以在处理系统中执行的函数、脚本或调用入口，其编解码方法由处理系统通过字符串前缀来约定。

> NOTE: 执行这些方法的方式，是由处理系统决定的。本规范的其它章节约定了处理方法的流程和接口，但处理系统可能在所有方面保持与PEDT规范一致，但在具体处理时采用不同的实现方案。例如task.map要求并行执行一组tasks，但实际实现中采用同步方案仍然是可行的——这仅仅带来性能上的差异而不影响对外的接口表现。
> 
> NOTE: 在特定的系统中的taskDef中，分发方法（作用于task）也可以表达为对象或编码字符串，这是该系统支持本地函数或对象支持（规范中推荐而非强制）所致。

方法中的处理方法包括taskDef.distributed、taskDef.promised和taskDef.rejected，分发方法包括task.run和task.map。其中，处理方法可以没有或有多个（它们当然不可重复的命名），一个拥有全部处理方法的典型的taskDef如下：

``` JSON
{
	"distributed": "script:lua:base64: ... ",
	"promised": "script:lua:base64: ... ",
  	"rejected": "script:lua:base64: ... "
}
```

一个任务必然有两个分发方法（task.run和task.map）之一。分发方法的任务参数(arguments)是可选定义的。此外，对于map方法来说，还必须要定义一个分发范围(scope)。一个典型的有map方法的task如下：

``` JSON
{
	"map": "task: ...",
	"scope": "...",
	"arguments": { ... }
}
```

一个典型的有run方法的task如下：

``` JSON
{
	"run": "task: ...",
	"arguments": { ... }
}
```

如下是实现了本地函数或对象支持（规范中推荐而非强制）的有run方法的task示例：

``` JSON
// 支持本地函数
{
	"run": "script:javascript:base64: ...",
	"arguments": { ... }
}
// 支持taskObject对象
{
	"run": { ... },
	"arguments": { ... }
}
```

##### 任务分发范围(distribution scope)

当一个task使用map方法时，它需要在task对象中定义一个scope成员来表达分发范围。该scope是一个未经编码一般字符串(normal string），采用三段标记（three parts token）格式：

``` text
<systemPart>:[pathPart]:<scopePart>
	systemPart  : 向一个系统分发
	pathPart    : 向上述系统的一个路径分发
	scopePart   : 向上述系统路径中的（一组结点的）范围分发
```

systemPart与scopePart不能包含":"字符，且不可以缺省；pathPart可以是（包括空字符串在内的）任意字符串。

> NOTE: 三段标记（three parts token）是固定分段的字符串，因此即使在pathPart中出现了":"字符，也不被作为分隔符处理。

正常scope字符串长度必然大于3。长度小于等于3的字符串被作为保留标识字(reserved tokens)。目前明确定义的保留标识字有：

``` text
>	"?"      : unhandled placeholder
>	"::*"	: unhandled distribution scope
>	"::?"	: distribution scope scopePart invalid
>	":::"	: distribution scope invalid
```

例如，如下是一个“带scope的map方法”的任务定义：

``` javascript
{
	"map": "task:570b41ba61ade63987d318b0c08e4fa4",
	"scope": "n4c:/a/b/c/test:*"
}
```

##### 任务参数（arguments）

task对象可以定义一个arguments成员来表达该任务执行时的依赖的参数，该参数（亦即该成员）必然是一个对象。多个参数用对象成员来表达，没有顺序关系。

> NOTE: 目前PEDT Specifications series 1并没有要求处理系统强制检查arguments的类型，这意味着它在具体实现时仍然可以是非对象的。但这仅限于在本地执行的task.run方法，且由于这对规范有明显的破坏性，因此会给“兼容或提升至”规范1.1的某些可选特性时带来障碍。

例如，如下是一个“带arguments的run方法”的任务定义：

``` javascript
{
	"run": "task:570b41ba61ade63987d318b0c08e4fa4",
	"arguments": { "arg1": 1, "arg2": true }
}
```

### 规范

#### PEDT define specification 0.9

> * 0.9-1 typeDef is limited JSON format
>   
>   ``` 
>   1. full support JSON value types
>   2. limited support JSON array and object types
>   	- array/object data can't using members or elements of > other array/object
>   	- task execute result is limited JSON format, or(optional) support full JSON format
>   ```
>   
> * 0.9-2 "task:" is top level prefix
>   
>   ``` 
>   1. task arguments is unsupported in task.run and task.map
>   2. local taskObject/function is unsupported in task.run
>   ```
>   
> * 0.9-3 "script:" prefix is optional
>   
>   ``` 
>   1. taskDef.distributed is optional
>   2. taskDef.promised is optional
>   3. taskDef.rejected is optional
>   ```

PEDT任务定义规范0.9的主要限制在于不支持JSON的array/object的多级定义或相互嵌套的定义。例如下面三个定义都是非法的：

``` javascript
// unsupported in 0.9
[[1]]
[{ "a": 1 }]
{ "a": [1] }
```

基于同样的原因，规范0.9也就不能支持标准的、对象格式的task。例如：

``` javascript
// unsupported in 0.9
{ "x": { "run": "task:570b41ba61ade63987d318b0c08e4fa4" } }
```

因为这里显然出现了多级的object。因此规范0.9要求用"task:"作为编码字符串前缀来表示这样一个task。

规范0.9中的taskDef不能使用数组（因为它必须是一个对象，且不支持嵌套），但在一个任务执行的返回结果中是可以支持数组的（即使是仅实现limited JSON format的版本）。此外，根据规范，返回结果是可选支持full JSON types的。

规范0.9可以用一个@前缀来表示taskDef.map中的scope。如果scope中包括":"字符或非utf8字符，那么整个task需要用base64（或其它约定的格式）来编码。类似如下的定义是合法的：

``` javascript
// supported in 0.9
{ "x": "task:570b41ba61ade63987d318b0c08e4fa4"}
{ "x": "task:570b41ba61ade63987d318b0c08e4fa4@localhost" }
{ "x": "task:base64:NTcwYjQxYmE2MWFkZTYzOTg3ZDMxOGIwYzA4ZTRmYTRAaHR0cDovL2xvY2FsaG9zdC90ZXN0OioK" }
```

> NOTE: "@localhost"中的localhost不是一个distribution scope，而是一个系统内部的范围标识(token)，所以它不是标准的three parts格式的。这是由不同处理系统决定的一个可选实现。

#### PEDT define specification 1.0

> * 1.0-1 full support JSON types
>   
> * 1.0-2 support top level prefix: "data:", "script:"
>   
>   ``` 
>     1. "task:" is not top level prefix, but support downward compatibility
>     2. "string:" is subType for "data:" only
>     3. support encodeType: "base64" and "utf8"
>   ```
>   
> * 1.0-3 support full taskDef/task features
>   
>   ``` 
>     1. support scope property for task.map method
>     2. support arguments property for task.map and task.run
>     3. support taskDef.promised, taskDef.distributed and taskDef.rejected fields
>     4. local taskObject/function is strong recommend in task.run
>   ```
>   
> * 1.0-4 support taskDef as member of other taskDef
>   
>   ``` 
>     1. support taskDef array as member of other taskDef
>   ```
>   
> * 1.0-5 typeDef as arguments is optional
>   
>   ``` 
>   1. "reduce" as task method is optional
>   2. "daemon" as task method is optional
>   ```

PEDT任务定义规范1.0的主要限制是task的arguments是一个简单的、未约定含义的JSON对象。因此是否能够将taskDef作为arguments，也就成了一个可选项。更进一步的带来了reduce/daemon等扩展方法也是可选项。

规范1.0是推荐实现reduce和方法的，但对如何实现没有明确约定。

#### PEDT define specification 1.1

> * 1.1-1 full features of specification 1.0 is supported, and downward compatibility
>   
> * 1.1-2 access current task processor in all taskDef methods and task.run method	
>   
> * 1.1-3 support taskDef as task arguments
>   
> * 1.1-4 more task method is optional
>   
>   ``` 
>     1. "reduce" and "daemon" as task method is strong recommend
>     2.  more task method is optional
>   ```
>   
> * 1.1-5 run method is map method at local, optional
>   
>   ``` 
>     1. support full/real distribution taskDef when this feature ready
>   ```

PEDT任务定义规范1.1具有目前已知的全部特性集，该规范强烈建议你实现reduce/daemon方法。并且，你可以将run方法作为一个本地的map方法来实现，一旦你这样做，则你整个的系统都是完整而纯粹的可分布式系统了。

> NOTE: 否则，你会有一部分任务将会因为使用了本地逻辑而不能“自由、随意”地迁移到其它结点上执行。然而，后者正是大多数系统的常态，要求“所有方法能能被分布”一定程度上也增加了不必要的系统负担。

最后，规范1.1是唯一一个要求你在任务方法中传递任务处理器(current task processor)的规范。它是当前正在使用的任务对象处理器实例（其实就是javascript中的this对象，或者lua中的self对象，等等类似于此）。

> NOTE: 这是依赖执行环境的，你并不一定能在一个非面向对象的环境中轻易地实现这一特性。

### keywords

JSON/JSON format

JSON values

object members/object fields

array elements

taskDef

static taskDef

distributed taskDef

published taskDef

local taskDef/unpublished taskDef

method

taskDef methods/process method

- distributed
- promised
- rejected

task methods/distribution method

- run
- map
- reduce
- daemon

local task

task/distribution task

task arguments

scope/distribution scope

three parts token

- systemPart
- pathPart
- scopePart

md5

taskId

taskId prefix

- "task:"

normal string

encoded string

encoded string prefix

three parts prefix

- "data:"
- "script:"
- "string:"
- "base64:"
- "utf8:"

current task processor/this/self

token

reserved tokens

## PEDT process specification

本规范用于描述如何处理taskDef，使得它可以在不同的执行环境中得到一致的处理。

### 基础定义

#### PEDT任务的四个处理阶段

* 任务声明：
  
  taskDef是一个任务的JSON格式文本，它用于存储和分发，它本身不是执行体（不可直接传递函数或代码）。任务中需要执行的部分，要么被编码(encoded)放在字符串中，要么作为一个外部的taskDef以它的taskId来引用（并声明在一个task中）。
  
* 任务预处理
  
  taskObject是上述taskDef解码后的对象。它首先是JSON文本到本地对象的解码，其次，还必须按PEDT的约定对其中的编码字符串解码。需要注意的是，编码字符串解码之后不一定仍然是字符串，而可能是本地支持的对象成员类型。例如"script:javascript"前缀的解码，就会使对应的成员被重写成函数。
  
  一个taskDef被注册后不可变更，因此它有一个唯一对应的taskId。同样，通过这样一个taskDef解码的本地对象taskObject也是不可变更的，这个对象可以被缓存，或者用作其它对象的原型。
  
* 任务订单
  
  taskOrder是taskObject的一个可处理映像，它可以是一个taskObject的子类或taskObject的一个实例。应确保taskOrder可以得到taskObject的全部可处理信息，且在处理过程中不会破坏taskObject。
  
  任务处理过程正式开始于taskOrder的提交。
  
* 任务结果
  
  taskResult是taskOrder被处理后的结果，它可以是taskOrder自身（taskOrder是可以改写的），或者是对taskOrder再次处理的、类型完全不同的结果。

#### PEDT任务的三个特定处理方法（process method）

在taskDef中可以声明三个处理方法，即taskDef.distributed、taskDef.promised和taskDef.rejected。

* distributed方法
  
  该方法是在当前环境被分发一个新的taskDef时调用的，它在当前环境中只执行一次。这意味着你可以在其中处理一些当前环境相关的信息。例如
  
  ``` text
  taskDef = { "node_id": "unknow" }
  ```
  
  这个声明中，node_id与当前执行环境相关，就可以在distributed方法中对它进行一次性赋值。
  
  由于distributed方法只执行一次，因此当前环境可以在该方法执行之后才转换成任务对象(taskObject)并缓存之。
  
* promised方法
  
  该方法是在taskOrder被正确处理并得到taskResult之后调用的。在该方法中你可以对taskResult做进一步的加工并返回，或者处理成其它结果值返回。
  
  如果promised方法没有返回值，则整个taskOrder仍以taskResult为结果；否则以promised返回值为结果。
  
  PEDT并不保证一个taskDef/taskOrder的执行结果与其原型（在对象类型上）一致。
  
* rejected方法
  
  该方法是在taskOrder处理中出现错误时调用的。在该方法中你可以对错误进行进一步的处置。一旦定义了该方法，则错误将被当前taskDef处理吞吃（mute），除非你再次显式地返回错误。
  
  在promised方法中处理taskResult出现错误时，也会调用rejected方法；默认情况下，rejected返回的值也被理解为一个成功返回的taskResult，但不会再将控制流程交回promised方法。

#### PEDT任务的两个特定分发方法（distribution method）

在task中可以声明两个分发方法，即task.run和task.map。一个任务有且有仅能只有一个分发方法。

> NOTE: 从接口的角度上来说，这两个方法是必须实现的。但在具体的环境中，它们实现为部分有效也是可行的，这取决于具体执行环境所处于的阶段（或状态）。所谓部分有效是指：接口是存在的，但处于不可用的状态。
> 
> NOTE: 在规范1.1的一个可选实现中，task.run是通过task.map来实现的，因此这种情况下处理系统内部将只有一个分发方法。

* run方法
  
  该方法意味着指定任务是在本地执行的，如果任务被指定为taskId，则处理系统应该先通过下载(download_task)来得到taskDef，并最终转换得到一个taskOrder提交执行。
  
* map方法
  
  该方法意味着指定任务是在异地的一组结点上执行的，并最终可以得到一组（相同数量的）执行结果，该组结果中每一个必然是一个taskResult。
  
  > NOTE: 由于PEDT规范并不保证taskResult与taskDef/taskOrder有对象类型上的相似性，因此上述一组结果可能是不相似的。

#### 对处理系统的基本需求

PEDT要求处理系统必须具备如下能力，这些能力简单地描述为可调用接口，以便处理系统具体实现之。

> NOTE: 各接口的具体说明参考本规范集中的“任务调用接口规范”部分。

* mix(Object obj, Object ref)
  
  处理系统应将ref对象的所有成员混入到obj对象上，返回结果是修改过的obj对象本身。
  
* Promise.all(Array arr)
  
  提供在map()中向远程RESTfull接口提交一组task并捕获返回结果的能力；并提供在taskDef中将一组任务本地执行并得到结果的能力。arr的成员为普通成员或promise对象。
  
  > NOTE: 关于Promise的细节参考：
  > 
  > * Promise in MDN: [https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
  > * Promises/A+ specification: [http://promisesaplus.com](http://promisesaplus.com)
  
* parse_scope(String scope)
  
  提供将distributionScope的部分或全部解析成一组可接受任务分发的结点的能力。这些结点的接口信息是在Promise.all()中处理远程调用时所依赖的。
  
* register_task(String taskDef)
  
  提供将taskDef放在远程的、所有结点可访问的公共结点中的能力。该接口将返回taskDef对应的taskId字符串。
  
* download_task(String taskId)
  
  提供从上述公共结点中获取由taskId指定的taskDef的能力。该接口将返回JSON值。

> NOTE: 以上三个接口通常被封装成internal_xxx接口，表明它们是内部实现的。并且，它们通常是调用外部的接口来实现功能，而自身只做接口与数据的转换。
> 
> NOTE: 这三个接口的具体实现与网络环境相关。

### 规范

#### PEDT process specification 1.0

1. 从taskDef到taskObject的过程
   
   前提：已经通过download_task(taskId)从远程得到taskDef的文本，或从本地缓存中得到taskDef。
   
   > 1.1 task_def_decode
   > 
   > > 1.1.1 转换成本地对象：localObj = JSON_decode(taskDef)
   > > 
   > > 1.1.2 处理每个成员，解码所有编码字符串：decodedObj = decode_task_fields(localObj)
   > > 
   > > > NOTE: decode_task_fields()处理localObj的每个编码字符串成员(obj.x)，并使obj.x = decode(obj.x)，且最终成员x可能不再是字符串类型。
   > > 
   > > 1.1.3 如果localObj的成员为对象，则为之递归调用1.1.2
   > > 
   > > 1.1.4 得到解码结果：taskObject = decodedObj
   > 
   > 1.2 distributed_task
   > 
   > 如果是第一次从远程得到taskDef，且存在distributed方法，调用该方法：distributed(taskObject)
   > 
   > 1.3 返回结果：return taskObject
   
2. 从taskOrder到taskResult的过程
   
   前提：已经创建了taskObject的一个映像作为taskOrder，确保修改taskOrder的成员不会影响taskObject，且可以访问后者的所有可访问成员。
   
   > 2.1 promise_static_member
   > 
   > > 2.1.1 处理taskOrder的每个成员，如果它是一个task对象，则为该task调用task.map或task.run方法；这些tasks记为taskOrder.x0..n。
   > > 
   > > > NOTE: 由于taskOrder初始为taskObject的一个映像，因此这时taskOrder.x0与taskObject.x0是相同的；又由于taskObject是从“一次分发，终身不变”的taskDef得来，所以这个x0..n的成员列表以及它们对应的tasks列表事实上也是不变的。
   > > 
   > > 2.1.2 将上述方法的结果作为promise对象放到一个与当前taskOrder相关联的数组promises中；这些promises记为promises[0..n]；
   > > 
   > > > NOTE: 可以在promises中放入被关联的taskOrder以便后续的处理过程能访问到它——而无需建立其它的关联或索引。
   > > 
   > > 2.1.3 调用Promise.all(promises)，并得到返回结果值results。如果在Promise.all()调用中出现异常，则进入2.4。
   > > 
   > > > NOTE: 按照Promise的规范，这个results实际是在.then()的onFulfilled函数中得到的。这个onFulfilled是all promises得到异步的、完全的确认——即执行并返回结果——之后一次性调用的。
   > 
   > 2.2 promise_member_rewrite
   > 
   > 按results的数组元素顺序，严格地将每一个result——这些result记为results[0..n]——抄写到（与之对应task成员的）成员中。即：
   > 
   > > taskOrder.x0 = results[0], ...
   > 
   > 2.3 pickTaskResult
   > 
   > > 2.3.1 得到结果值：taskResult = taskOrder
   > > 
   > > 2.3.2 如果taskDef.promised存在，则为taskOrder调用一次该方法：taskResult = promised(taskResult)。如果在调用promised(taskResult)中出现异常，则进入2.4，否则进入2.5。
   > 
   > 2.4 rejected
   > 
   > > 2.4.1 taskResult = taskDef.rejected(reason)
   > 
   > 2.5 返回结果：return taskResult
   
3. 实现execute_task(String taskId, Object args)
   
   该方法要求处理系统执行由taskId指定的taskDef，处理系统可以从本地缓存装载或远程下载该taskDef，也可以执行由taskDef得到并缓存的taskObject，但最终需要返回taskResult。
   
   > 3.1 internal_download_task
   > 
   > 从taskId得到一个可用的taskObject。
   > 
   > > 3.1.1 调用内部的download_task()从taskId得到taskDef
   > > 
   > > 3.1.2 调用本规范之process 1，从taskDef得到taskObject
   > 
   > 3.2 internal_execute_task
   > 
   > 需要在执行taskDef之前处理args。
   > 
   > > 3.2.1 从taskObject得到一个可写映像作为taskOrder，然后mix(taskOrder, args)
   > > 
   > > > NOTE: 这意味着args中的成员值可以影响到当前处理的taskDef（的映像）的成员信息。
   > > 
   > > 3.2.2 调用本规范之process 2，从taskOrder得到taskResult
   > > 
   > > > NOTE: 在规范1.1中“taskDef可以用作arguments”的约定不影响本接口。亦即是说，这里的args不会得到“预先作为taskDef加以执行”的机会。
   > 
   > 3.3 返回结果：return taskResult
   
4. 实现map(String distributionScope, String taskId, Object args)
   
   处理系统应先将distributionScope解释成一组可接受任务分发的结点，并向这些结点的execute_task接口投放RESTfull请求，这些请求中会将args作为参数传入。
   
   处理系统应在执行调用远程的execute_task接口之前处理args。
   
   > NOTE: 在规范1.1中，该args可以是一个新的taskDef2，这种情况下taskDef2会被先执行，且taskDef2的结果对象将作为taskDef的输入参数。
   > 
   > NOTE: taskId和args作为RESTfull请求中的参数（或数据）传递的细节参考本规范之“调用接口规范”。
   
   处理系统应从上述结点的execute_task接口中获取返回结果，并将结果按调用顺序装入数组，最终将数组结果解析成本地可用的数据，以作为本次map()调用的返回结果。
   
   > 4.1 internal_parse_scope
   > 
   > 将distributionScope解释成request_uri[0..n]数组。
   > 
   > > NOTE: 对distributionScope的解释方法参考本规范之“调用接口规范”。
   > 
   > 4.2 distributed_request
   > 
   > 将request_uri[]提交为一组远程的、并发的WEB RESTfull请求。
   > 
   > > 4.2.1 假设request_uri[1..n]记录每个可接受任务分发的结点的uri，该uri指向目标结点中用WEB RESTfull规格颂的execute_task()接口。则distributed_request()用于向该uri发送
   > > 
   > > > http.request(uri + taskId)
   > > 
   > > 这样的GET/POST请求。同时，args将可以http请求中的参数提交到每一个结点。
   > > 
   > > 4.2.2 用Promise.all()等待所有请求返回结果。
   > > 
   > > > NOTE: Promise.all()会将每一个结果装入与request_uri[0..n]对应的results[0..n]。
   > 
   > 4.3 extractMapedTaskResult
   > 
   > 从results抽取结果值：maped = JSON_decode(results[0..n])
   > 
   > > NOTE: 这里的语义相当于JavaScript中的：
   > > 
   > > ``` javascript
   > > maped = results.map(JSON.parse)
   > > ```
   > > 
   > > NOTE: execute_task()的RESTfull接口将返回JSON值，因此本地系统总是能将它解析成对象并放入数组，且最终结果数组仍可以序列化为JSON值。
   > 
   > 4.4 返回结果数组：return maped
   
5. 实现run(task, Object args)
   
   处理系统将针对task的不同情况做处理。
   
   处理系统应在执行task之前处理args。
   
   > 5.1 如果task是一个taskId字符串，则交由execute_task()来处理。参见本规范之处理process 3 - 实现execute_task(String taskId, Object args)。
   > 
   > 5.2 如果task是一个本地函数，则使用args对象作为唯一参数调用之。
   > 
   > 5.3 如果task是一个对象，则尝试作为一个本地任务对象taskObject调用之。参见本规范之处理process 3.2 - internal_execute_task。
   > 
   > > NOTE: 在规范1.1中，args参数可以是一个新的taskDef2，这种情况下taskDef2会被先执行，且taskDef2的结果对象将作为taskDef的输入参数。
   > > 
   > > NOTE: 5.2和5.3是强烈推荐实现(strong recommend)，但不是必需实现的。
   > > 
   > > NOTE: 过程5.2与其它两个处理在对待args上并不相同。5.2是将args作为函数的唯一参数调用，而5.1和5.3是调用mix()将args混入到taskOrder。
   > > 
   > > NOTE: 对于过程5.3，当task是一个taskObject时，即使它拥有（从taskDef继承来的）distributed方法，也不会被调用。

## PEDT interface specification

### 本地接口

本地接口是指：需要在当前处理系统中实现的接口，以便处理系统完成本规范所定义的上述处理过程。

#### 混入对象：mix

``` javascript
funtion mix(obj, ref)
 - 参数：
	obj: JSON_Supported，混入目标对象
	ref: JSON_Supported，参考的混入源（对象或值）
 - 返回值：
	return: 如果obj不是对象，则返回一个混入了ref的新对象；否则返回混入了ref的obj自身；如果ref无效(undefined/nil等)，则总是返回obj。
```

说明：

如果调用是ref无效，则返回值是obj，这种情况下并不能确保总是一个对象（因为obj可能是值而不是对象）。

如果obj是对象，则mix()总是返回obj自身，只是它的成员被混入了ref的成员。其中与ref同名的成员被重写，在obj中不存在的成员将从ref中抄写。混入运算是递归的。

> NOTE: JSON_Supported在这里是指所有JSON支持的数据类型。ref和obj是JSON值parse得到的对象，因此其成员限定为4种值类型和两个引用类型(复合类型，对象和数组)。

#### HTTP请求：distributed_request

``` javascript
function distributed_request(URLs, taskId, args)
 - 参数：
	URLs: Array of String, 一组结点的服务地址。
	taskId: String, 以"task:"为前缀的字符串。
	args: Object，参数对象。
 - 返回值：
	taskResults: Array of HTTP_Responsed，一组远程执行taskId后的结果数组。
```

说明：

处理器在distributed_request()中实现对于一组服务地址(URLs)的分布式请求。这些请求被封装成两种可能的格式：HTTP GET/POST。

无论是使用GET或POST请求，taskId总是作为请求URL的一部分追加到URLs[x]后面。例如（注意其中的taskId是带字符串前缀"task:"的）:

> ``` text
> http://.../execute_task:570b41ba61ade63987d318b0c08e4fa4
> ```

或：

> ``` text
> http://.../call?task:570b41ba61ade63987d318b0c08e4fa4
> ```

在使用GET请求时，args参数将会被编码成url参数继续追加到上面的URL。从args对象到url参数字符串编码的方法，参考：

> [querystring.stringify in nodejs](https://nodejs.org/api/querystring.html#querystring_querystring_stringify_obj_sep_eq_options)
> 
> [ngx.encode_args in lua-nginx](https://github.com/openresty/lua-nginx-module#ngxencode_args)

在使用POST请求时，你有可以选择如下两种HTTP headers之一，来设定在body/data区传送args的方法。

``` text
Content_Type: application/x-www-form-urlencoded
Content_Type: application/json
```

当使用x-www-form-urlencoded格式时，应该将args按上述url参数编码的方法编码，并作为post data提交；当使用json格式时，应该将args序列化成json文本，并作为post data提交。如果Content_Type缺省，则服务端应以“application/x-www-form-urlencoded”作为默认格式处理。

在使用POST请求时，由于url中也可以同时传递参数，因此服务端最终解析得到的args将会是url paraments与body data混合的结果。这种情况下，你可以通过在Content_Type中加入参数，以指示服务端解析中忽略混入(mixin)操作。例如：

``` text
Content_Type: application/x-www-form-urlencoded; mixin=false
Content_Type: application/json; mixin=false
```

你仍然可以在Content_Type中加入language或其它参数来指示服务端的其它行为。

最后，distributed_request()在处理服务端的返回结果时，HTTP_Responsed是指具体实现者通过HTTP协议返回的某个结构或数据，这与具体实现的方法有关。通常是一个HTTP Response对象，且response.header存放返回结果的头，response.body存放返回结果的数据区。

#### 方法：taskDef.distributed

``` javascript
function taskDef.distributed(taskObject)
 - 参数：
	taskObject: Object, 由taskDef解码得到的对象
 - 返回值：无
```

说明：

distributed是taskDef的一个可选声明的处理方法。它可以在taskDef下载到本地之后对它做一些小的修改，例如添加本地标识、IP地址或重写一些task的参数等等。

distributed通常用于改写taskObject，但也可以用来在得到这个taskDef时执行一些初始化的任务，例如创建本地服务等等。

#### 方法：taskDef.promised

``` javascript
function taskDef.promised(taskResult)
 - 参数：
	taskResult: Object, 重写后的taskOrder
 - 返回值：未确定类型, 返回任务的执行结果taskResult或任意可能的值
```

说明：

promised()是taskDef的一个可选声明的处理方法。它可以在每次taskDef执行时得到一次处理返回值(taskResult)的机会。在promised()中可以修改taskResult，或返回新的值。甚至，也可以在promised()中调用新的分发方法。

如果promised()有非null的返回值，则使用该返回值作为当前taskOrder的执行结果；否则仍然以taskResult作为执行结果——无论它是否在promise()中修改过。

> NOTE: 不能在promised()中直接返回结果值null，因为这会被理解为“使用当前taskResult”。但在具体的实现环境中，也可以通过返回一个promise的方法来达到相同的效果。例如：
> 
> ``` javascript
> return Promise.resolve(null)
> ```

#### 方法：taskDef.rejected

``` javascript
function taskDef.rejected(reason)
 - 参数：
	reason: JSON_Supported, 一个描述错误的值或对象
 - 返回值：未确定类型, 返回任务的执行结果taskResult或任意可能的值
```

说明：

rejected()是taskDef的一个可选声明的处理方法。它可以在每次taskDef执行出错时得到一次处置机会。在rejected()中可以构造并返回一个新的taskResult值，或继续触发reject。

也可以在rejected()中调用新的分发方法，并返回后者的结果；如果在rejected()中出现错误reason2，处理系统将忽略原有错误，而以新的reason2返回错误。

如果rejected()有非null的返回值，则使用该返回值作为当前taskOrder的正确执行结果；否则它仍然以reason作为一个出错的结果。如果用户代码自行reject新的（或既有的）reason，则也将是一个出错的结果。

> NOTE 1: 如何reject新的reason是由处理系统决定的。例如在javascript中的：
> 
> ``` javascript
> return Promise.reject(reason)
> ```
> 
> NOTE 2: 与taskDef.promised()相同，在具体的处理系统中，用户代码也可以返回null值。例如：
> 
> ``` javascript
> return Promise.resolve(null)
> ```

#### 解析范围

``` javascript
function internal_parse_scope(distributionScope)
 - 依赖：参考本规范中“远程接口 - 资源query”
 - 参数：
	distributionScope: String, 三段标记（three parts token）字符串
 - 返回值：
	return: Array of String, 字符串数组，是scope对应的一组结点的RESTfull接口地址
```

说明：

internal_parse_scope接受三段标记（systemPart:pathPart:scopePart），但只将其中的systemPart:pathPart两节作为标识提交到远程查询(query)并得到一组结点的接口地址URLs。

> NOTE: 在具体的实现版本中，通常通过require+subscribe机制来封装query接口而非直接调用之，以避免重复的distributionScope解析。

随后，internal_parse_scope在本地通过scopePart的定义对URLs进行过滤，以得到最终返回的字符串数组。

PEDT为scopePart仅预留了一个"*"表达式，表明是URLs的全集。但scopePart也可以是其它的值，以表明还需要通过其它特殊处理来从URLs筛选出一个子集。一些可能的示例包括：

``` text
"free>2g"：表示过滤所有结点中当前空闲内存>2G的结点
"incluce('master')"：表示过滤所有结点名中包括master符串的结点
```

这些scopePart表达式由处理系统自行约定与实现。

#### 下载任务

``` javascript
function internal_download_task(taskId)
 - 依赖：参考本规范中“远程接口 - 任务下载”
 - 参数：
	taskId: String, 以"task:"为前缀的字符串。
 - 返回值：
	return: String, taskDef的JSON文本。
```

说明：

处理系统应根据与远端（例如任务注册中心结点）协商的接口，通过taskId得到与之对应的taskDef。

> NOTE: 如何向远端注册一个任务，与本规范中的任务执行过程是无关的。

#### 执行任务

``` javascript
function execute_task(taskId, args)
 - 依赖：参考本规范中“本地接口 - 下载任务”
 - 参数：
	taskId: String, 以"task:"为前缀的字符串。
	args: Object, 本地对象，其成员应当是JSON支持的数据类型
 - 返回值：未确定类型, 返回任务的执行结果taskResult或任意可能的值
```

说明：

处理系统会先将taskId转换至taskOrder，然后调用mix()将args混入到taskOrder，然后再开始执行这个taskOrder。

执行结果是数据类型未定义的。缺省情况下它应当是将被rewrite的taskOrder作为taskResult，但经过taskDef.promised()的处理之后，它可能是任意值或对象。

#### 方法：task.run

``` javascript
function task.run(task, args)
 - 依赖：参考本规范中“本地接口 - execute_task”
 - 参数：
	task: String/Function/Object, 以"task:"为前缀的字符串，或函数，或对象。
	args: Object, 本地对象，其成员应当是JSON支持的数据类型
 - 返回值：未确定类型, 返回任务的执行结果taskResult或任意可能的值
```

说明：

如果task是一个taskId字符串，则交由execute_task()来处理。

> NOTE: 以下为强烈推荐实现(strong recommend)的部分
> 
> * 如果task是一个本地函数，则使用args对象作为唯一参数调用之。
> * 如果task是一个对象，则尝试作为一个本地任务对象taskObject调用之。

执行结果是数据类型未定义的。当task是一个有效的taskId或对象时，在缺省情况下返回值应当是将被rewrite的taskOrder作为taskResult，但经过taskDef.promised()的处理之后，它可能是任意值或对象。当task是本地函数时，是该函数的返回值。

#### 方法：task.map

``` javascript
function task.map(distributionScope, taskId, args)
 - 依赖：参考本规范中“本地接口 - internal_parse_scope”
 - 依赖：参考本规范中“本地接口 - HTTP请求：distributed_request”
 - 依赖：参考本规范中“远程接口 - 任务执行/execute_task”
 - 参数：
	distributionScope: String, 三段标记（three parts token） 
	taskId: String, 以"task:"为前缀的字符串。
	args: Object, 本地对象，其成员应当是JSON支持的数据类型
 - 返回值：
	return: Array, 总是一个数组，但成员的个数和成员的类型都具有不确定性。
```

说明：

处理器需要调用internal_parse_scope()来将distributionScope中的systemPart:pathPart解析成一组可访问的结点地址，并假设这些结点能接受远程的execute_task请求，这些请求约定为RESTfull调用。

处理器需要在本地解析distributionScope中的scopePart，以确定向上述结点地址列表中的部分或全部发出请求。

> NOTE: 由于scopePart动态调整了可访问结点的范围，因此并不能确保发出请求的数量（是否等于结点地址列表的大小），同样，也就不能确保相应的结果数组的成员个数。

处理器将taskId作为RESTfull请求URL的一部分，使用distributed_request接口（HTTP请求）的方式向上述结点发出请求。这些请求可能是GET的，或POST的，以便能在调用时携带args作为参数或请求数据。

处理将上述请求的返回结果作为JSON文本处理，然后解析成本地对象并放入结果数组。最后，返回结果数组作为task.map()的返回值。

> NOTE: 由于远程的execute_task接口并不保证返回数据与taskId所对应的taskDef/taskObject在数据类型上一致，因此task.map尽管必然返回一个与发出请求数组相同大小、相同顺序的结果数组，但不能确保结果数组中的成员数据类型一致。

### 远程接口

远程接口是指：需要在当前处理系统之外实现的接口，当前处理系统默认这些远程接口已经先于服务请求调用之前就绪，并且都是HTTP RESTfull接口的形式交付。

在以下接口描述中，

``` 
* 使用curl的方式来描述请求，
* "jq ."用于在控制台上显示返回的JSON结果，
* ${SERV}参数指代提供服务的地址。
```

#### 任务注册/register_task

``` bash
> curl -s -XPOST --data-binary @taskDef.json "${SERV}/register_task?version=1.1" | jq .
"task:68bb82e2a6bcbb5f9a83b93c85cff07a"
```

可以用version参数来指定taskDef采用的规范版本，当该参数缺省时，服务端默认为1.1。

成功返回时，其结果是一个JSON字符串(有""引号）而非普通文本。推荐使用“Content-Type: application/json”来返回成功调用的结果。

> NOTE: 客户端可以通过检查返回文本中否有'"task:'前缀来判断成功与否。

当服务端遭遇错误时，推荐通过如下方法来返回错误信息：

> ``` text
> * 置http_status_code为5xx；
> * 置header中的Content-Type为"application/json"；
> * 将错误信息文本作为json body返回。
> ```

#### 任务下载/download_task

``` bash
> curl -s -D- "${SERV}/download_task:68bb82e2a6bcbb5f9a83b93c85cff07a" 
HTTP/1.1 200 OK
Content-Type: application/json
X-Pedt-Version: 1.1
Date: Wed, 28 Oct 2015 02:59:10 GMT
Content-Length: 68

{
	"x": { "run": "script:lua:utf8:function() return 'Hi!' end" }
}
```

实际发出的请求Path为"/download_"，"task:"被作为前缀的一部分补在上述URL Path之后。在该GET请求返回的header中，用“X-Pedt-Version”来表示该taskDef的注册版本，缺省为1.1。

请求的发起者在获得taskDef文本后，可根据“X-Pedt-Version”来自行判断是否支持该任务。

成功调用时，返回结果是一个符合PEDT规范的、JSON格式的taskDef文本。推荐使用“Content-Type: application/json”来返回成功调用的结果。

> NOTE: 客户端可以通过检查返回文本中否有'{'前缀来判断成功与否。

当服务端遭遇错误时，通过如下方法来返回错误信息：

> ``` 
> * 置http_status_code为5xx；
> * 置header中的Content-Type为"application/json"；
> * 将错误信息文本作为json body返回。
> ```

#### 任务执行/execute_task

``` bash
> curl -s  "${SERV}/execute_task:68bb82e2a6bcbb5f9a83b93c85cff07a" | jq .
{
	"x": "Hi!"
}
```

实际发出的请求Path为"/execute_"，"task:"被作为前缀的一部分补在上述URL Path之后。

参考本规范之“本地接口 - HTTP请求：distributed_request”，你可以在请求的URL中添加更多的参数（即execute_task的args参数），或使用POST请求并提交data。更进一步的，在使用POST请求时，也可以通过header来指定data的上下文类型(Context_Type)或指示参数args的处理方法。

成功调用时，返回结果可能是任意JSON文本。推荐使用“Content-Type: application/json”来返回成功调用的结果。

> NOTE: 客户端可以通过检查返回文本前缀，以确定返回值是否是一个有效的JSON文本（注意这并非绝对可靠的方法）。

当服务端遭遇错误时，通过如下方法来返回错误信息：

> ``` 
> * 置http_status_code为5xx；
> * 置header中的Content-Type为"application/json"；
> * 将错误信息文本作为json body返回。
> ```

#### 资源query

``` bash
> curl -s  "${SERV}/query?SYS:a/b/c/d" | jq .
[
  "http://127.0.0.1:8011/kwaf/invoke?execute=",
  "http://127.0.0.1:8012/kwaf/invoke?execute=",
  "http://127.0.0.1:8010/kwaf/invoke?execute=",
  "http://127.0.0.1:8013/kwaf/invoke?execute="
]
```

在使用GET请求时，实际发出的请求Path为"../query"，其后的整个search字符串"SYS:a/b/c/d"是distributionScope中的systemPart:pathPart两节。

如果采用POST请求，则应发送如下JSON对象作为请求数据：

``` JSON
{
	"key": "SYS:a/b/c/d",
	"type": "scope",
	"version": "1.1"
}
```

当key值缺省时，服务端会将URL中的search作为key（同于GET请求）。

无论是在GET/POST请求中，key总是使用encodeURI()编码过的。

成功调用时，服务端的返回结果总是一个字符串数组。可以为空数组。推荐使用“Content-Type: application/json”来返回成功调用的结果。

> NOTE: 客户端可以通过检查返回文本中否有'['前缀来判断成功与否。

当服务端遭遇错误时，通过如下方法来返回错误信息：

> ``` 
> * 置http_status_code为5xx；
> * 置header中的Content-Type为"application/json"；
> * 将错误信息文本作为json返回。
> ```

### 其它

以下列出的接口是推荐实现的。

#### 资源订阅/subscribe

``` bash
> curl -s -XPOST --data '{...}' "${SERV}/subscribe" | jq .
[
  "http://127.0.0.1:8011/kwaf/invoke?execute=",
  "http://127.0.0.1:8012/kwaf/invoke?execute=",
  "http://127.0.0.1:8010/kwaf/invoke?execute=",
  "http://127.0.0.1:8013/kwaf/invoke?execute="
]
```

在资源服务中提供query接口的同时，也可以为指定systemPart:pathPart提供订阅(subscribe)接口。这种情况下，systemPart:pathPart被理解为resource key。

subscribe接口只支持POST请求，且它的post data与“资源查阅/query”基本一致：

``` JSON
{
	"key": "SYS:a/b/c/d",
	"type": "scope",
	"version": "1.1",
	"notify": "..."
}
```

在上述post data中的notify字段用于在服务端保留一个订阅者的列表，当被订阅资源发生变更时，订阅者会在notify地址上得到一个通知，订阅者应视为resource key对应的资源失效并发起重取。

notify上的通知与上述“资源查阅/query”采用完全相同的协议，即：将systemPart:pathPart作为GET请求的search string，或作为POST请求中的key字段。

subscribe接口返回与query相同的结果——事实上会在完成订阅之后调用query。

#### 资源请求/require

对远端接口query的一个简单封装，例如实现本地缓存等。它总是返回与query相同的结果。

require接口用于在多级的资源管理服务中实现代理(proxy)，避免客户端直接面临query+subscribe接口。

#### 方法task.reduce

``` javascript
function task.reduce(distributionScope, taskId, reduce)
或
function task.reduce(distributionScope, taskId, args, reduce)
 - 参数：
	distributionScope, taskId, args: (参见task.map)
	reduce: function, 在task.map行为之后的回调函数
 - 返回值：未确定类型，是调用task.run(reduce, taskResults)之后的结果。
```

本方法是经典的map/reduce模型的一个实现，由接口中的reduce函数来处理map()返回的一组结果taskResults。

在没有args参数时，实现为如下逻辑(示例采用javascript描述)：

``` javascript
return task.run(reduce, task.map(distributionScope, taskId))
```

在有args参数时，实现为如下逻辑(示例采用javascript描述)：

``` javascript
return task.run(reduce, task.map(distributionScope, taskId, args))
```

#### 方法task.daemon

``` javascript
function task.daemon(distributionScope, taskId, daemon, daemonArgs)
 - 参数：
	distributionScope, task: (参见task.map)
	daemon: function, 在task.map行为之前调用的函数
	daemonArgs: Object, 使用task.run()来调用daemon时传入的参数
 - 返回值：Array，是调用task.map()之后的结果。
```

本方法先在本地调用task.run(daemon, daemonArgs)，以确保本地启动一个服务（或由本地launch一个远程服务），然后再调用task.map()。这个map调用的返回值通常不重要，因为真正的结果或行为的对象是上述的、已经预先启动的服务。

该方法总是实现为：

``` javascript
return task.map(distributionScope, taskId, task.run(daemon, daemonArgs))
```

> NOTE: 注意这里是将daemon运行/启动返回结果值作为map()中的args传入。因此map不能自行再定义args值——即使真的有这种需要，也应当定义在daemonArg中并由daemon来返回给map()。

# 实现

(暂略)

# Q&A

1. 为什么会有规范0.9
   
   规范0.9的核心是在不支持过于复杂的文本分析或面向对象的环境中使用PEDT任务，并且确保能被更高版本的规范向下兼容。比如，在写定这一规范的过程中我们充分地参考了它的一个在BASH环境中的实现。
   
2. 规范0.9真的不支持使用任务参数吗
   
   的确。但是你可以在处理系统中通过两阶段地执行任务来实现它。这是指：首先从一个远程的或本地的配置系统中取得参数，并在处理系统中准备好全局(或局部的)变量，然后执行task并使用这些全局(或局部的)变量。
   
3. 规范1.0是作为弱化的1.1版本是为什么存在的
   
   规范1.0的核心是静态任务定义与执行。它包含完整的任务定义特性，并能很容易地在跨系统环境得到处理，但是它有关动态执行的所有特性都是“可选的或未定义的”，这意味着要试图在一个任务中动态地发起一个新任务是很难的。
   
   规范1.0很适合交付静态逻辑的任务系统，例如定时触发的任务，数据采集与预处理任务，或特定的watch任务。但很难应对有动态逻辑的、复杂的分布式任务的业务。
   
4. 为什么string使用三段前缀
   
   它的简单的好处是可以在忽略编码的情况下，通过正向查找快速地确定内容(context/body)的类型，例如当前环境只需要检查“script:lua:”即可确定是否处理。而编码类型是在前缀的后半部分成对、嵌套声明的，因此可以简单递归以完成解码，而无需考虑前半部分的类型声明。
   
5. 处理系统是否需要实现完整的Promise
   
   PEDT规范要求按照MDC中的规范（事实上是ECMAScript的规范）来实现Promise，但这并非是强制性的。
   
   首先，PEDT事实上只要求了以Promise.all接口的形式来实现并行任务，从接口上来说，处理系统对此实现成异步的或同步的并不会导致接口上的差异（而只有性能上的不同）。其次，PEDT规范中，单个处理结点对外有且仅有一个execute_task接口，该接口只有RESTfull的接口规接要求，并没有限制其内部实现是否使用Promise，或是否是同步/异步。
   
   所以处理系统有能力在当前结点内部使用全同步的方式来实现PEDT并对外暴露execute_task接口，以表明自己能够处理PEDT任务。这在基于规范0.9的系统中，是可行的（甚至是常见的）策略。在这样的处理方案中，Promise.all被一个同步的任务队列处理替代，而其它类似的异步特性（例如task.run）也采用了类似处理方法。
   
   从另一个角度上来看，处理系统也可以实现Promise.all的异步效果而无需实现Promise框架，这在技术上仍然是可行的。
   
6. 为什么使用"/execute_"这样的RESTfull接口
   
   由于PEDT在使用简单的GET请求时需要通过url来传递参数args，因此不适合将taskId作为一个url参数(paraments)来处理。所以在制定本规范的接口时描述时，建议将taskId作为Url Path的一部分追加到接口上。
   
   如果使用POST请求，由于可以将args作为data提交，并在header中添加mixin=false来避免服务端将args混入url paraments，因此这种情况下也可以将taskId作url参数来编码。
   
   在RESTfull接口的地址（URL+Path）的设计上，本规范是指导性而非强制性的。例如在n4c架构中，ngx_cc模块实现对PEDT协议的支持时，这里使用的URL就是'http://.../../invoke?execute='。
   
7. 可以使用检查前缀的方法来远程接口返回值的有效性吗
   
   我们不推荐这种方法。
   
   所有PEDT的返回均默认为application/json类型：
   
   > * 可以是符合JSON格式的值，而并不一定是对象）；
   > * 在返回错误时是通过http status来标识的，而并没有（强制地）限定response body的内容。
   
   在这样的情况下，使用前缀检测并不安全有效。但是以下情况总是失败的（反之并不能确定总是成功的）：
   
   > * 调用register_task而返回结果不以『"task:』前缀开始。
   > * 调用download_task而返回结果不以“{”开始。
   > * 调用task.map()或资源require()而返回结果不以“[”开始。