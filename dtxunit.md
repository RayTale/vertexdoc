## Vertex提供强一致性事务的目的主要是以牺牲性能来简化代码逻辑，使逻辑代码更清晰更易维护

#### 使用强一致性事务功能会降低单个Actor的吞吐能力，但对系统的整体吞吐能力影响不超过20%。

* 以转账功能为例，单个用户不能高频转账和到账，但是同时可能有几万几十万的账户Actor在互相转账，这种场景强调的是整体吞吐能力，对单个用户账户的Actor的吞吐要求不高，这种场景的最终一致性和强一致性其实吞吐差距不大，这时候就可以使用强一致性来简化逻辑代码。

  

* 以股票交易为例，某只股票的Actor因为需要处理几十万用户提交的订单，是一对多的关系，所以单个Actor的吞吐能力就非常重要，这种场景就不适合强一致性。
* 如果追求极致性能，那建议所有场景都使用最终一致性

#### 以转账为例，实现一个完整的强一致性功能

##### 1、AccountActor：账户Actor

* 接口定义

``` csharp
    public interface IDTxAccount : IVertexActor, IDTxActor, IGrainWithIntegerKey
    {
        //转账扣款
        Task<bool> Transfer(long toAccountId, decimal amount);
        //转账到账
        Task TransferArrived(decimal amount);
    }
```

* 快照定义

``` csharp
    public class AccountSnapshot : ITxSnapshot<AccountSnapshot>
    {
        public decimal Balance { get; set; }

        public AccountSnapshot Clone(ISerializer serializer)
        {
            return new AccountSnapshot
            {
                Balance = this.Balance
            };
        }
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
  ```

* 实现

``` csharp
 public class DTxAccount : DTxActor<long, AccountSnapshot>, IDTxAccount
    {
        public async Task<bool> Transfer(long toAccountId, decimal amount)
        {
            if (Snapshot.Data.Balance >= amount)
            {
                var evt = new TransferEvent
                {
                    Amount = amount,
                    Balance = Snapshot.Data.Balance - amount,
                    ToId = toAccountId
                };
                await TxRaiseEvent(evt);
                return true;
            }
            else
                return false;
        }
        public Task TransferArrived(decimal amount)
        {
            var evt = new TransferArrivedEvent
            {
                Amount = amount,
                Balance = Snapshot.Data.Balance + amount
            };
            return TxRaiseEvent(evt).AsTask();
        }
    }
```

##### 2、TransferDtxUnitActor：事务管理器Actor

* 接口定义

``` csharp
    public interface ITransferDtxUnit : IDTxUnitActor<TransferRequest, bool>, IGrainWithIntegerKey
    {
    }
```

* 事务消息定义(请求事务管理器执行事务的消息)

``` csharp
    public class TransferRequest
    {
        public string Id { get; set; }
        public long FromId { get; set; }
        public long ToId { get; set; }
        public decimal Amount { get; set; }
    }
```

* 实现

``` csharp
  public class TransferDtxUnit : DTxUnitActor<long, TransferRequest, bool>, ITransferDtxUnit
    {
        public override string FlowId(TransferRequest request) => request.Id;
        //事务逻辑
        public override async Task<bool> Work(TransferRequest request)
        {
            try
            {
                var result = await GrainFactory.GetGrain<IDTxAccount>(request.FromId).Transfer(request.ToId, request.Amount);
                if (result)
                {
                    await GrainFactory.GetGrain<IDTxAccount>(request.ToId).TransferArrived(request.Amount);
                    //提交事务
                    await Commit();
                    return true;
                }
                else
                {
                    //回滚事务
                    await Rollback();
                    return false;
                }
            }
            catch
            {
                //回滚事务
                await Rollback();
                throw;
            }
        }
        //事务中受影响的Actor
        protected override IDTxActor[] EffectActors(TransferRequest request)
        {
            return new IDTxActor[]
            {
                GrainFactory.GetGrain<IDTxAccount>(request.FromId),
                GrainFactory.GetGrain<IDTxAccount>(request.ToId),
            };
        }
    }
```

* 调用

``` csharp
var actor = client.GetGrain<ITransferDtxUnit>(1);
var result = actor.Ask(new TransferRequest
            {
                FromId = account,
                ToId = account + accountCount,
                Amount = 50,
                Id = idGen.CreateId().ToString()
            })
```
