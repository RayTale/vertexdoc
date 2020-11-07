## 以转账流程为例，实现一个完整的最终一致性流程

#### 1、AccountActor：账户Actor，里面有用户的余额信息，并提供充值到账、转账扣款、转账到账和转账回退的方法。

* 接口定义

``` csharp
    public interface IAccount : IVertexActor, IGrainWithIntegerKey
    {

        /// <summary>
        /// 转账扣款方法
        /// </summary>
        /// <param name="toAccountId">转账的目标账户Id</param>
        /// <param name="amount">转账金额</param>
        /// <param name="transferId">转账Id</param>
        /// <returns></returns>
        Task<bool> Transfer(long toAccountId, decimal amount, string transferId);
        /// <summary>
        /// 转账到账方法
        /// </summary>
        /// <param name="amount">Amount to account</param>
        /// <param name="transferId">转账Id(可空，上下文Id会自动传递)</param>
        /// <returns></returns>
        Task TransferArrived(decimal amount, string transferId);
        /// <summary>
        /// Refund for failed transfer
        /// </summary>
        /// <param name="amount">amount</param>
        /// <param name="transferId">转账Id(可空，上下文Id会自动传递)</param>
        /// <returns></returns>
        Task<bool> TransferRefunds(decimal amount, string transferId);
    }
```

* 快照定义

  

``` csharp
   public class AccountSnapshot : ISnapshot
    {
        //账户余额
        public decimal Balance { get; set; }
    }
  ```

* 事件定义

    

``` csharp
    //转账事件
    [EventName(nameof(TransferEvent))]
    public class TransferEvent : IEvent
    {
        public long ToId { get; set; }
        public decimal Amount { get; set; }
        public decimal Balance { get; set; }
    }
    //转账到账事件
    [EventName(nameof(TransferArrivedEvent))]
    public class TransferArrivedEvent : IEvent
    {
        public decimal Amount { get; set; }
        public decimal Balance { get; set; }
    }
    //转账回退事件
    [EventName(nameof(TransferRefundsEvent))]
    public class TransferRefundsEvent : IEvent
    {
        public decimal Amount { get; set; }
        public decimal Balance { get; set; }
    }
  ```

* 实现

  

``` csharp
  public sealed class Account : VertexActor<long, AccountSnapshot>, IAccount
    {

        public Task<bool> Transfer(long toAccountId, decimal amount, string transferId)
        {
            //余额判断
            if (Snapshot.Data.Balance >= amount)
            {
                var evt = new TransferEvent
                {
                    Amount = amount,
                    Balance = Snapshot.Data.Balance - amount,
                    ToId = toAccountId
                };
                return RaiseEvent(evt, transferId);
            }
            else
                return Task.FromResult(false);
        }

        public Task TransferArrived(decimal amount, string transferId)
        {
            var evt = new TransferArrivedEvent
            {
                Amount = amount,
                Balance = Snapshot.Data.Balance + amount
            };
            return RaiseEvent(evt,transferId);
        }

        public Task<bool> TransferRefunds(decimal amount, string transferId)
        {
            var evt = new TransferRefundsEvent
            {
                Amount = amount,
                Balance = Snapshot.Data.Balance + amount
            };
            return RaiseEvent(evt,transferId);
        }
    }
  ```

#### 2、AccountFlowActor: 自动监听AccountActor的事件，并执行EventHandle启动后续流程

* 接口定义

``` csharp
    public interface IAccountFlow : IFlowActor, IGrainWithIntegerKey
    {
    }
```

* 实现

  

``` csharp
    public sealed class AccountFlow : FlowActor<long>, IAccountFlow
    {
        readonly IGrainFactory grainFactory;
        public AccountFlow(IGrainFactory grainFactory)
        {
            this.grainFactory = grainFactory;
        }
        public override IVertexActor Vertex => grainFactory.GetGrain<IAccount>(this.ActorId);
        public Task EventHandle(TransferEvent evt)
        {
            if(验证成功)
            {
                //转账成功
                var toActor = GrainFactory.GetGrain<IAccount>(evt.ToId);
                return toActor.TransferArrived(evt.Amount,evt.TransferId);
            }else
            {
                //转账回退
                var fromActor = GrainFactory.GetGrain<IAccount>(evt.FromId);
                return fromActor.TransferRefunds(evt.Amount,evt.TransferId);
            }
        }
    }
  ```

  #### 3、AccountDbActor: 自动监听AccountActor的事件，把事件信息同步到数据库(数据库数据有延时，只能用来查询)

* 接口定义

``` csharp
    public interface IAccountDb : IFlowActor, IGrainWithIntegerKey
    {
    }
```

* 实现

  

``` csharp
    public sealed class AccountDb : FlowActor<long>, IAccountDb
    {
        readonly IGrainFactory grainFactory;
        public AccountDb(IGrainFactory grainFactory)
        {
            this.grainFactory = grainFactory;
        }
        public override IVertexActor Vertex => grainFactory.GetGrain<IAccount>(this.ActorId);

        public Task EventHandle(TransferEvent evt, EventMeta eventBase)
        {
            //进行数据库修改操作
            return Task.CompletedTask;
        }
        public Task EventHandle(TopupEvent evt, EventMeta eventBase)
        {
            //进行数据库修改操作
            return Task.CompletedTask;
        }
        public Task EventHandle(TransferArrivedEvent evt, EventMeta eventBase)
        {
            //进行数据库修改操作
            return Task.CompletedTask;
        }
        public Task EventHandle(TransferRefundsEvent evt, EventMeta eventBase)
        {
            //进行数据库修改操作
            return Task.CompletedTask;
        }
    }
  ```
