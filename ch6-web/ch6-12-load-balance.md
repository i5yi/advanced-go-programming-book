# 6.12. Load-Balance 负载均衡

本节将会讨论常见的 web 后端服务之间的负载均衡手段。

## 常见的负载均衡思路

如果我们不考虑均衡的话，现在有 n 个 endpoint，我们完成业务流程实际上只需要从这 n 个中挑出其中的一个。有几种思路:

1. 按顺序挑: 例如上次选了第一台，那么这次就选第二台，下次第三台，如果已经到了最后一台，那么下一次从第一台开始。这种情况下我们可以把 endpoint 都存储在数组中，每次请求完成下游之后，将一个索引后移即可。在移到尽头时再移回数组开头处。

2. 随机挑一个: 每次都随机挑，真随机伪随机均可。设选择第 x 台机器，那么 x 可描述为 `rand.Intn() % n`。

3. 根据某种权重，对下游 endpoints 进行排序，选择权重最大/小的那一个。

当然了，实际场景我们不可能无脑轮询或者无脑随机，如果对下游请求失败了，我们还需要某种机制来进行重试，如果纯粹的随机算法，存在一定的可能性使你在下一次仍然随机到这次的问题节点。

我们来看一个生产环境的负载均衡案例。

## 一种随机负载均衡算法

考虑到我们需要随机选取每次发送请求的 endpoint，同时在遇到下游返回错误时换其它节点重试。所以我们设计一个大小和 endpoints 数组大小一致的索引数组，每次来新的请求，我们对索引数组做洗牌，然后取第一个元素作为选中的服务节点，如果请求失败，那么选择下一个节点重试，以此类推:

```go
var endpoints = []string {
    "100.69.62.1:3232",
    "100.69.62.32:3232",
    "100.69.62.42:3232",
    "100.69.62.81:3232",
    "100.69.62.11:3232",
    "100.69.62.113:3232",
    "100.69.62.101:3232",
}

// 重点在这个 shuffle
func shuffle(slice []int) {
    for i := 0; i < len(slice); i++ {
        a := rand.Intn(len(slice))
        b := rand.Intn(len(slice))
        slice[a], slice[b] = slice[b], slice[a]
    }
}

func request(params map[string]interface{}) error {
    var indexes = []int {0,1,2,3,4,5,6}
    var err error

    shuffle(indexes)
    maxRetryTimes := 3

    idx := 0
    for i := 0; i < maxRetryTimes; i++ {
        err = apiRequest(params, indexes[idx])
        if err == nil {
            break
        }
        idx++
    }

    if err != nil {
        // logging
        return err
    }

    return nil
}

```

我们循环一遍 slice，两两交换，这个和我们平常打牌时常用的洗牌方法类似。看起来没有什么问题。

## 有没有什么问题？

真的没有问题么？实际上还是有问题的。这段简短的程序里有两个隐藏的隐患:

1. 没有随机种子。在没有随机种子的情况下，rand.Intn 返回的伪随机数序列是固定的。

2. 洗牌不均匀，会导致整个数组第一个节点有大概率被选中，并且多个节点的负载分布不均衡。

第一点比较简单，应该不用在这里给出证明了。关于第二点，我们可以用概率知识来简单证明一下。假设

## 修正后的负载均衡算

## zk 集群的随机节点挑选问题