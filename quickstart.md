* Vertex的主体是一个集群服务，主要业务功能运行在集群中，通过API对外提供服务。
* Vertex通过构建Client对象来访问集群的API。
> 我们通过转账案例来构建一个完整的服务
---

# Cluster
Cluster的集群化配置请参照Orleans的集群设置
#### 接口项目(IGrains)
* 创建一个名称为Transfer.IGrains的项目
* 通过nuget引入Vertex.Abstractions包
* 项目中创建IAccount接口，表示用户金额账户的API，代码如下
  ```csharp
  public interface IAccount : IVertexActor, IGrainWithIntegerKey
    {
        /// <summary>
        /// 查询当前余额
        /// </summary>
        /// <returns></returns>
        [AlwaysInterleave]
        Task<decimal> GetBalance();

        /// <summary>
        /// 充值金额，方便进行测试
        /// </summary>
        /// <param name="amount">金额</param>
        /// <param name="topupId">充值Id</param>
        /// <returns></returns>
        Task<bool> TopUp(decimal amount, string topupId);

        /// <summary>
        /// 转账操作
        /// </summary>
        /// <param name="toAccountId">目标账户Id/param>
        /// <param name="amount">转账金额</param>
        /// <param name="transferId">转账的唯一ID</param>
        /// <returns></returns>
        Task<bool> Transfer(long toAccountId, decimal amount, string transferId);

        /// <summary>
        /// 转账到账方法
        /// </summary>
        /// <param name="amount">到账金额</param>
        /// <returns></returns>
        Task TransferArrived(decimal amount);

        /// <summary>
        /// 转账失败金额回退方法
        /// </summary>
        /// <param name="amount">回退的金额</param>
        /// <returns></returns>
        Task<bool> TransferRefunds(decimal amount);
  ```
* 创建一个IAccountDb接口，这个是一个空接口，主要用来监听事件，然后把金额变化同步到数据库，方便查询
  ```csharp
    public interface IAccountDb : IFlowActor, IGrainWithIntegerKey
    {
    }
  ```

* 创建一个IAccountFlow接口，这个是一个空接口，主要用来监听事件，然后启动后续业务流程
  ```csharp
    public interface IAccountFlow : IFlowActor, IGrainWithIntegerKey
    {
    }
  ```
#### 接口实现项目(Grains)
* 创建一个名称为Transfer.Grains的项目
* 引入以下nuget包:
  > Vertex.Storage.Linq2db:基于linq2db的存储实现，支持postgresql、mysql、sqlserver、sqlite等数据库

  > Vertex.Stream.Common:事件流相关的公共实现，用来进行事件分发

  > Vertex.Runtime:运行时库
* 引用项目Transfer.IGrains
* 定义事件类
  ```csharp
    //充值事件
    [EventName(nameof(TopupEvent))]//事件名称设置
    public class TopupEvent : IEvent
    {
        //充值金额
        public decimal Amount { get; set; }

        //充值后的余额
        public decimal Balance { get; set; }
    }
    //转账事件
    [EventName(nameof(TransferEvent))]
    public class TransferEvent : IEvent
    {
        //转账的目标id
        public long ToId { get; set; }
        //转账金额
        public decimal Amount { get; set; }
        //转账后的余额
        public decimal Balance { get; set; }
    }
    //转账到账事件
    [EventName(nameof(TransferArrivedEvent))]
    public class TransferArrivedEvent : IEvent
    {
        //到账的金额
        public decimal Amount { get; set; }
        //到账后的余额
        public decimal Balance { get; set; }
    }
    //转账回退事件
    [EventName(nameof(TransferRefundsEvent))]
    public class TransferRefundsEvent : IEvent
    {
        //回退的金额
        public decimal Amount { get; set; }
        //回退后的余额
        public decimal Balance { get; set; }
    }
  ```

* 实现IAccount接口
  ```csharp
    [EventStorage(Consts.CoreDbName, nameof(Account), 3)]// 事件存储相关的配置
    [EventArchive(Consts.CoreDbName, nameof(Account), "month")]// 事件归档的配置，如果事件不进行归档，可以不设置
    [SnapshotStorage(Consts.CoreDbName, nameof(Account), 3)]// 状态快照相关的配置
    [Stream(nameof(Account), 3)]//事件流的配置
    public sealed class Account : VertexActor<long, AccountSnapshot>, IAccount
    {
        //查询余额
        public Task<decimal> GetBalance()
        {
            return Task.FromResult(this.Snapshot.Data.Balance);
        }

        public Task<bool> Transfer(long toAccountId, decimal amount, string transferId)
        {
            if (this.Snapshot.Data.Balance >= amount)//如果余额足够，则进行转账
            {
                var evt = new TransferEvent
                {
                    Amount = amount,
                    Balance = this.Snapshot.Data.Balance - amount,
                    ToId = toAccountId
                };
                return this.RaiseEvent(evt, transferId);
            }
            else
            {
                return Task.FromResult(false);
            }
        }
        //金额充值、方便测试
        public Task<bool> TopUp(decimal amount, string topupId)
        {
            var evt = new TopupEvent
            {
                Amount = amount,
                Balance = this.Snapshot.Data.Balance + amount
            };
            return this.RaiseEvent(evt, topupId);
        }
        
        //转账金额到账
        public Task TransferArrived(decimal amount)
        {
            var evt = new TransferArrivedEvent
            {
                Amount = amount,
                Balance = this.Snapshot.Data.Balance + amount
            };
            return this.RaiseEvent(evt);
        }
        //转账金额回退(流程终止)
        public Task<bool> TransferRefunds(decimal amount)
        {
            var evt = new TransferRefundsEvent
            {
                Amount = amount,
                Balance = this.Snapshot.Data.Balance + amount
            };
            return this.RaiseEvent(evt);
        }
    }
  ```
* 实现IAccountFlow接口
  ```csharp
    [SnapshotStorage(Consts.CoreDbName, nameof(AccountFlow), 3)]//快照保存设置(快照用来记录事件的消费情况)
    [StreamSub(nameof(Account), "flow", 3)]//监听的事件流设置，监听的事件会送到这里进行消费

    public sealed class AccountFlow : FlowActor<long>, IAccountFlow
    {
        //这里设置事件源的地址，方便事件丢失的时候从这里拉取
        public override IVertexActor Vertex => this.grainFactory.GetGrain<IAccount>(this.ActorId);
        //定义个方法，用来接收TransferEvent事件
        public Task EventHandle(TransferEvent evt, EventMeta eventBase)//eventBase为可选，如果不需要就不用声明
        {
            var toActor = this.GrainFactory.GetGrain<IAccount>(evt.ToId);
            //调用充值的目标账户，进行到账操作
            return toActor.TransferArrived(evt.Amount);
            //这里也可以进行转账回退操作
        }
    }
  ```
* 实现IAccountDb接口
  ```csharp
    [SnapshotStorage(Consts.CoreDbName, nameof(AccountDb), 3)]
    [StreamSub(nameof(Account), "db", 3)]
    public sealed class AccountDb : FlowActor<long>, IAccountDb
    {
        public override IVertexActor Vertex => this.GrainFactory.GetGrain<IAccount>(this.ActorId);

        public Task EventHandle(TransferEvent evt, EventMeta eventBase)
        {
            // Update database here
            return Task.CompletedTask;
        }

        public Task EventHandle(TopupEvent evt)
        {
            // Update database here
            return Task.CompletedTask;
        }

        public Task EventHandle(TransferArrivedEvent evt, EventMeta eventBase)
        {
            // Update database here
            return Task.CompletedTask;
        }

        public Task EventHandle(TransferRefundsEvent evt)
        {
            // Update database here
            return Task.CompletedTask;
        }
  ```

#### 启动项目(Host)
创建一个控制台项目，代码如下
```csharp
public class Program
    {
        public static Task Main(string[] args)
        {
            var host = CreateHost();
            return host.RunAsync();
        }

        private static IHost CreateHost()
        {
            return new HostBuilder()
                .UseOrleans(siloBuilder =>
                {
                    siloBuilder
                        .UseLocalhostClustering()
                        .Configure<ClusterOptions>(options =>
                        {
                            options.ClusterId = "dev";
                            options.ServiceId = "Transfer";
                        })
                        .Configure<EndpointOptions>(options => options.AdvertisedIPAddress = IPAddress.Loopback)
                        .ConfigureApplicationParts(parts =>
                        {
                            parts.AddApplicationPart(typeof(Account).Assembly).WithReferences();
                            parts.AddApplicationPart(typeof(DIDActor).Assembly).WithReferences();
                            parts.AddApplicationPart(typeof(StreamIdActor).Assembly).WithReferences();
                        })
                        .AddSimpleMessageStreamProvider("SMSProvider", options => options.FireAndForgetDelivery = true).AddMemoryGrainStorage("PubSubStore");
                })
                .ConfigureServices(serviceCollection =>
                {
                    serviceCollection.AddVertex();
                    serviceCollection.AddLinq2DbStorage(
                        config =>
                    {
                        config.Connections = new Vertex.Storage.Linq2db.Options.ConnectionOptions[]
                        {
                         //这里使用SQLite内存数据库，方便无依赖启动
                         new Vertex.Storage.Linq2db.Options.ConnectionOptions
                         {
                            Name = Consts.CoreDbName,
                            ProviderName = "SQLite.MS",
                            ConnectionString = "Data Source=Vertex.SQLite.db;"
                         }
                        };
                    }, new EventArchivePolicy("month", (name, time) => $"Vertex_Archive_{name}_{DateTimeOffset.FromUnixTimeSeconds(time).ToString("yyyyMM")}".ToLower(), table => table.StartsWith("Vertex_Archive".ToLower())));
                    //这里使用内存消息队列，方便无依赖启动
                    serviceCollection.AddInMemoryStream();
                })
                .ConfigureLogging(logging =>
                {
                    logging.SetMinimumLevel(LogLevel.Information);
                    logging.AddConsole();
                }).Build();
        }
    }
```

# Client
客户端用于访问集群和对外提供api，是外部流量进入集群的中转服务，可以是asp.net core、Grpc等等

我们这里创建一个简单的控制台程序，来讲解怎么访问cluster

```csharp
//创建客户端对象，可以使用依赖注入注入到容器中，这样其它地方只需要注入IClusterClient就可以访问集群
private static async Task<IClusterClient> StartClientWithRetries(int initializeAttemptsBeforeFailing = 5)
{
    int attempt = 0;
    IClusterClient client;
    while (true)
    {
        try
        {
            var builder = new ClientBuilder()
                .UseLocalhostClustering()
                .ConfigureApplicationParts(parts =>
                    parts.AddApplicationPart(typeof(IAccount).Assembly).WithReferences())
                .ConfigureLogging(logging => logging.AddConsole());
            client = builder.Build();
            await client.Connect();
            Console.WriteLine("Client successfully connect to silo host");
            break;
        }
        catch (Exception)
        {
            attempt++;
            Console.WriteLine(
                $"Attempt {attempt} of {initializeAttemptsBeforeFailing} failed to initialize the Orleans client.");
            if (attempt > initializeAttemptsBeforeFailing)
            {
                throw;
            }

            await Task.Delay(TimeSpan.FromSeconds(5));
        }
    }

    return client;
}
//启动入口
private static async Task Main(string[] args)
{
    using var client = await StartClientWithRetries();
    var txId = Guid.NewGuid().ToString();//唯一id，代表这次充值的唯一id，会在流程中传播，保证整个流程的幂等性
    var account_1 = client.GetGrain<IAccount>(1);
    var account_2 = client.GetGrain<IAccount>(2);
    await account_1.TopUp(100, txId);//先给账户1充值100
    var transferId = Guid.NewGuid().ToString();//转账唯一id
    await account_1.Transfer(2, 50, transferId);//账户1给账户2转账50元
    await Task.Delay(1000);//流程是异步的，所以需要等待完成才能获取到正确的余额
    Console.WriteLine($"账户1的余额={await account_1.GetBalance()}");
    Console.WriteLine($"账户2的余额={await account_2.GetBalance()}");
｝
```