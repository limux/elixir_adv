3-GenServer
===========

>本章讲述的GenServer不论在Erlang还是Elixir中都是非常核心的内容。
GenServer是OTP提供的一个抽象物，对OTP编程中经常用到的一个事物---服务器模型其中的通用部分进行了封装，从而程序员不需要重复实现这些通用部分，而是实现关键的行为代码。该行为代码页分两部分，用户API和操作该GenServer的回调方法。详细内容请开始阅读。

>GenServer是一个Elixir模块，在使用时用```use```导入。该知识点可以参考即将推出的《Elixir元编程》内容。

[第一个GenServer]()  
[测试一个GenServer]()  
[需要监控]()  
[call，cast还是info？]()  
[监视器还是链接？]()  

上一章我们用agent实现了buckets，而根据第一章所描述的，我们的设计是给每个bucket赋予一个名字：
```
CREATE shopping
OK

PUT shopping milk 1
OK

GET shopping milk
1
OK
```

因为agent是个进程，因而每个bucket首先有一个进程的id（pid）而不是名字。
回忆在《入门》中的进程那章，我们学习过给进程注册名字。
鉴于此，我们可以使用这个方法来给bucket起名：
```elixir
iex> Agent.start_link(fn -> [] end, name: :shopping)
{:ok, #PID<0.43.0>}
iex> KV.Bucket.put(:shopping, "milk", 1)
:ok
iex> KV.Bucket.get(:shopping, "milk")
1
```

这可是个很差的主意！在Elixir中，这些名字都会存储为原子。这意味着我们从外部客户端输入的bucket名字，都会被转换成原子。
记住，__绝对不要把用户输入转换为原子__。这是因为原子是不会被垃圾收集器收集。一旦原子被创建，它就不会被撤下（你也没法主动释放一个原子，对吧）。使用用户输入生成原子就意味着用户可以插入足够不同的名字来耗尽系统内存空间！
在实际操作中，在它用完内存之前会先触及Erland虚拟机的最大原子数量，从而造成系统崩溃。

比起滥用名字注册机制，我们可以创建我们自己的_注册表进程_来维护一个字典，用该字典联系起每个bucket的名字和进程。

这个注册表要能够保证永远处于最新状态。如果有一个bucket进程因故崩溃，注册表必须清除该进程信息，以防止继续服务下次查找请求。
在Elixir中，我们描述这种情况会说“该注册表进程需要监视（monitor）每个bucket进程”。  

我们将使用一个GenServer来创建一个注册表进程用来监视bucket进程。
在Elixir和OTP中，GenServer是创建这样进程的首选抽象物。

## 3.1-第一个GenServer

一个GenServer实现分为两个部分：用户API和服务器回调函数。这两部分都要写在同一个模块里。
下面我们创建文件```lib/kv/registry.ex```，包含以下内容：
```elixir
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry.
  """
  def start_link(opts \\ []) do
    GenServer.start_link(__MODULE__, :ok, opts)
  end

  @doc """
  Looks up the bucket pid for `name` stored in `server`.

  Returns `{:ok, pid}` if the bucket exists, `:error` otherwise.
  """
  def lookup(server, name) do
    GenServer.call(server, {:lookup, name})
  end

  @doc """
  Ensures there is a bucket associated to the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  ## Server Callbacks

  def init(:ok) do
    {:ok, HashDict.new}
  end

  def handle_call({:lookup, name}, _from, names) do
    {:reply, HashDict.fetch(names, name), names}
  end

  def handle_cast({:create, name}, names) do
    if HashDict.get(names, name) do
      {:noreply, names}
    else
      {:ok, bucket} = KV.Bucket.start_link()
      {:noreply, HashDict.put(names, name, bucket)}
    end
  end
end
```

第一个函数是```start_link/1```，它启动了一个新的GenServer。
其调用GenServer模块的```start_link/3```函数所使用的三个参数：  

1. 定义和实现了服务器回调函数的模块名称。这里的```__MODULE__```是当前模块名
2. 初始参数，这里是```:ok```
3. 一个选项列表，它可以存放服务器的名字等

你可以向一个GenServer发送两种请求。__Call__是同步的，服务器__必须__发送回复给该类请求。
__Cast__是异步的，服务器__不会__发送回复消息。

再往下的两个方法，```lookup/2```和```create/2```，它们用了两种不同方式发送请求给服务器。
这两种请求，会被第一个参数所指认的服务器中的```handle_call/3```和```handle_cast/2```函数处理（因此你的服务器回调函数必须包含这两个函数）。```GenServer.call/2```和```GenServer.cast/2```除了指认服务器之外，还告诉服务器它们要发送的请求。
这个请求存储在元组里，这里即```{:lookup, name}```和```{:create, name}```，在下面写相应的回调处理函数时会用到。
这个消息元组第一个元素一般是要服务器做的事儿，后面的元素就是该动作的参数。

在服务器这边，我们要实现一系列服务器回调函数来实现服务器的启动、停止以及处理请求等。
回调函数是可选的，我们在这里只实现所关系的那几个。

第一个是```init/1```回调函数，它接受一个状态参数（你在用户API中调用```GenServer.start_link/3```中使用的那个），返回```{:ok, state}```。这里```state```是一个新建的```HashDict```。
我们现在已经可以观察到，GenServer的API中，客户端和服务器之间的界限十分明显。```start_link/3```在客户端发生。
而其对应的```init/1```在服务器端运行。

对于```call```请求，我们在服务器端必须实现```handle_call/3```回调函数。参数：接收某请求（那个元组）、请求来源(```_from```)以及当前服务器状态（```names```）。```handle_call/3```函数返回一个```{:reply, reply, new_state}```形式的元组。其中，```reply```是你要回复给客户端的东西，而```new_statue```是新的服务器状态。

对于```cast```请求，我们必须实现一个```handle_cast/2```回调函数，接受参数：```request```以及当前服务器状态（```names```）。这个函数返回```{:noreply, new_state}```形式的元组。

这两个回调函数，```handle_call/3```和```handle_cast/2```还可以返回其它几种形式的元组。还有另外几种回调函数，如```terminate/2```和```code_change/3```等。可以参考[完整的GenServer文档](http://elixir-lang.org/docs/stable/elixir/GenServer.html)来学习相关知识。

现在，来写几个测试来保证我们这个GenServer可以执行预期工作。

## 3.2-测试一个GenServer
测试一个GenServer和测试agent比没有多少不同。我们在测试的setup回调中启动该服务器进程用以测试。
用以下内容创建测试文件```test/kv/registry_test.exs```：
```elixir
defmodule KV.RegistryTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, registry} = KV.Registry.start_link
    {:ok, registry: registry}
  end

  test "spawns buckets", %{registry: registry} do
    assert KV.Registry.lookup(registry, "shopping") == :error

    KV.Registry.create(registry, "shopping")
    assert {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

    KV.Bucket.put(bucket, "milk", 1)
    assert KV.Bucket.get(bucket, "milk") == 1
  end
end
```

哈，居然都过了！

关闭这个注册表进程，我们只需在测试结束时简单地发送```:shutdown```信号给该进程（别忘记，GenServer也是个进程而已，发送结束进程信号就可以粗暴地停止它）。这个方法对于测试是还好啦，只是，最好你还是在GenServer的处理逻辑里加上关于停止的方法。
比如，定义一个```stop/1```函数来发送要求停止的```call```请求，就是极好的：
```elixir
## Client API

@doc """
Stops the registry.
"""
def stop(server) do
  GenServer.call(server, :stop)
end

## Server Callbacks

def handle_call(:stop, _from, state) do
  {:stop, :normal, :ok, state}
end
```
上面代码中，新的```handle_call/3```回调函数专门处理```:stop```请求。它返回的元组包括一个原子```:stop```，后面跟着原因```:normal```，然后是```:ok```和服务器状态。

## 3.3-需要监控

至此，我们的注册表完成的差不多了。剩下的问题是这个注册表在有bucket崩溃的时候会失去时效。
比如增加一个一下测试来暴露这个问题：
```elixir
test "removes buckets on exit", %{registry: registry} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(registry, "shopping")
  Agent.stop(bucket)
  assert KV.Registry.lookup(registry, "shopping") == :error
end
```

这个测试会在最后一个断言处失败。因为当我们停止了bucket进程后，该bucket名字还存在于注册表中。

为了解决这个bug，我们需要注册表能够监视它派生出的每一个bucket进程。一旦我们创建了监视器，注册表将收到每个bucket退出的通知。
这样它就可以清理bucket映射字典了。

我们先在命令行中玩弄一下监视机制。启动```iex -S mix```：
```elixir
iex> {:ok, pid} = KV.Bucket.start_link
{:ok, #PID<0.66.0>}
iex> Process.monitor(pid)
#Reference<0.0.0.551>
iex> Agent.stop(pid)
:ok
iex> flush()
{:DOWN, #Reference<0.0.0.551>, :process, #PID<0.66.0>, :normal}
```

注意```Process.monitor(pid)```返回一个唯一的引用，使我们可以通过这个引用找到其指代的监视器发来的消息。
在我们停止agent之后，我们可以用```flush()```函数刷新所有消息，此时会收到一个```:DOWN```消息，内含一个监视器返回的引用。它表示有个bucket进程退出，原因是```:normal```。

现在让我们重新实现下服务器回调函数。
首先，将GenServer的状态改成两个字典：一个用来存储```name->pid```映射关系，另一个存储```ref->name```关系。
然后在```handle_cast/2```中加入监视器，并且实现一个```handle_info/2```回调函数用来保存监视消息。
下面是修改后完整的服务器调用函数：
```elixir
## Server callbacks

def init(:ok) do
  names = HashDict.new
  refs  = HashDict.new
  {:ok, {names, refs}}
end

def handle_call({:lookup, name}, _from, {names, _} = state) do
  {:reply, HashDict.fetch(names, name), state}
end

def handle_call(:stop, _from, state) do
  {:stop, :normal, :ok, state}
end

def handle_cast({:create, name}, {names, refs}) do
  if HashDict.get(names, name) do
    {:noreply, {names, refs}}
  else
    {:ok, pid} = KV.Bucket.start_link()
    ref = Process.monitor(pid)
    refs = HashDict.put(refs, ref, name)
    names = HashDict.put(names, name, pid)
    {:noreply, {names, refs}}
  end
end

def handle_info({:DOWN, ref, :process, _pid, _reason}, {names, refs}) do
  {name, refs} = HashDict.pop(refs, ref)
  names = HashDict.delete(names, name)
  {:noreply, {names, refs}}
end

def handle_info(_msg, state) do
  {:noreply, state}
end
```

我们没有修改任何客户端API而是稍微修改了服务器的实现。这就体现出了GenServer将客户端与服务器隔离开的好处。

最后，不同于其他回调函数，我们定义了一个“捕捉所有消息”的```handle_info/2```的函数子句（可参考《入门》，其意类似重载的函数的一条实现）。它丢弃那些不知道也用不着的消息。下面一节来解释下WHY。

## 3.4-call，cast还是info？

到目前为止，我们已经使用了三个服务器回调函数：```handle_call/3```，```handle_cast/2```和```handle_info/2```。何时使用哪个，其实很直白：  
1. ```handle_call/3```用来处理同步请求。这是默认的处理方式，因为等待服务器回复是十分有用的“反向压力（backpressure，涉及IO优化，请自行搜索）”机制。
2. ```handle_cast/2```用来处理异步请求，当你无所谓要不要个回复时。一个cast请求甚至不保证服务器收到了该请求，因此请有节制地使用。例如，我们定义的```create/2```函数应该使用call的，而我们用cast只是为了演示目的。
3. ```handle_info```用来接收和处理服务器收到的既不是```GenServer.call/3```也不是```GenServer.cast/2```的请求。它可以接受是以普通进程身份通过```send/2```收到的消息或者其它消息。监视器发来的```:DOWN```消息就是个极好的例子。

因为任何消息，包括通过```send/2```发送的消息，回去到```handle_info/2```处理，因此便会有很多你不需要的消息跑进服务器。
如果不定义一个“捕捉所有消息”的函数子句，这些消息会导致我们的监督者进程（supervisor）崩溃，因为没有函数子句匹配它们。

我们不需要为```handle_call/3```和```handle_cast/2```担心这个情况，因为它们能接受的请求都是通过GenServer的API发送的，要是出了毛病就是程序员自己犯错。

## 3.5-监视器还是链接？
我们之前在_进程_那章里的学习过链接（links）。现在，随着注册表的完工，你也许会问：我们啥时候用监控器，啥时候用链接呢？

链接是双向的。你将两个进程链接起来，其中一个挂了，另一个也会挂（除非它处理了该异常，改变了行为）。
而监视机制是单向的：只有监视别人的进程会收到被监视的进程的消息。
简单说，当你想让某些进程一挂都挂时，使用链接；而想要得到进程退出或挂了等事件的消息通知，使用监视。

回到我们```handle_cast/2```的实现，你可以看到注册表是同时链接着且监视着派生出的bucket：
```elixir
{:ok, pid} = KV.Bucket.start_link()
ref = Process.monitor(pid)
```

这是个坏主意。我们不想注册表进程因为某个bucket进程挂而一同挂掉！我们将在讲解监督者（supervisor）时探索更好的解决方法。
一句话概括，我们将不直接创建新的进程，而是将把这个责任委托给监督者。
就像我们即将看到的那样，监督者同链接工作在一起，这就解释了为啥基于链接的API（如```spawn_link```，```start_link```等）在Elixir和OTP上十分流行。

在讲监督者之前，我们首先探索下使用GenEvent进行事件管理以和处理的知识。
