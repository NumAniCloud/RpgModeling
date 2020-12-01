# パッシブ修飾システム

## 生成元

```csharp
[PassiveProcess("BattleContext")]
class PassiveEffectSettings
{
    // ActorAbility内のint, bool, float, stringなどのプリミティブ型が対象
    ActorAbility Ability;

    // 必ずBefore, Afterが両方作られる
    AttackBattleEvent AttackBattleEvent;
    DamageBattleEvent DamageBattleEvent;
}
```

## 概念

* ドメインイベント / DomainEvent
* ドメインコンテキスト / DomainContext
* パッシブ処理 / PassiveProcess
* 補正付きデータ型 / FinalData
* ドメインイベントハンドラー / DomainEventHandler
* パッシブ処理フックハンドラー / PassiveProcessHookHandler

## 生成範囲についての考察

実際に利用している様子のクラス図は以下の通り。

![](out/Passive/Semantics.png)

`<<user>>`, `<<generated>>` とついているものはライブラリに含めることはできない。
しかし、他のクラスはライブラリに用意することができるだろうか？

それらのクラスは `user/generated` なクラスに依存しているため、単純ではない。

### IBattleEvent -> IPassiveProcessProvider

`IPassiveProcessProvider` という概念を廃止し、
`IEnumerable<PassiveProcess>` を持てばよいかもしれない。

その場合 `IBattleEvent` はジェネリクス型 `IDomainEvent<T>` となる。
バトル画面以外のドメインでは、
Tの部分にバトル画面以外のためのパッシブ処理が入るので自然かもしれない。

### BattleEventHandler -> PassiveProcessHookHandler

`BattleEventHandler` をジェネリクス型 `DomainEventHandler<T>` として扱い、
`PassiveProcessHookHandler` は新たなインターフェース `IPassiveProcessHookHandler<PassiveProcess>` を実装する形にするとよいかもしれない。

`DomainEventHandler<T>` は `IPassiveProcessHookHandler<T>` に依存するだけとなり、
そういったクラスはライブラリに含めることができる。

### まとめ

以下のような実装にできそう。
ちなみに、 `IEnumerable<T>` は標準ライブラリの型。

![](out/Passive/Semantics2.png)

型引数 `T` は、そのクラス階層の属するドメインの特徴を記述するものとして考えられる。
例えば、 `BattlePassive` なる名前のクラスは恐らく戦闘ドメインに属するだろう。

library

* IDomainEvent&lt;T>
* DomainEventHandler&lt;T>
* IPassiveProcessHookHandler&lt;T>

user

* BattleContext
* Ability
* ConcretePassiveProcess
* ConcreteBattleEvent
* Battler

generated

* **PassiveProcess**
* Final**Ability**
* **PassiveProcess**HookHandler

## イベント内容の実装

以下のように、イベント内容自体も型にできるかもしれない。

```csharp
using ProcessFunc = IPassiveProcessFunction<PassiveProcess>;

abstract class PassiveProcess
{
    public virtual IEnumerable<ProcessFunc> LeadingProcesses { get; }
    public virtual IEnumerable<ProcessFunc> FollowingProcesses { get; }
    public virtual int ModifyAttack(int source) => source;
    public virtual int ModifyDefence(int source) => source;

    protected PassiveProcessFunction<PassiveProcess, TEvent> Create<TEvent>(
        Func<TEvent, BattleContext, Task> processFunc)
        where TEvent : IBattleEvent<PassiveProcess>
    {
        return new PassiveProcessFunction<PassiveProcess, TEvent>(processFunc);
    }
}

interface IPassiveProcessFunction<TPassive>
{
    Task RunAsync(IBattleEvent<TPassive> @event, BattleContext context);
}

class PassiveProcessFunction<TPassive, TEvent> : IPassiveProcessFunction<TPassive>
    where TEvent : IBattleEvent<TPassive>
{
    private readonly Func<TEvent, BattleContext, Task> _processFunc;

    public PassiveProcessFunction(Func<TEvent, BattleContext, Task> processFunc)
    {
        _processFunc = processFunc;
    }

    public async Task RunAsync(IBattleEvent<TPassive> @event, BattleContext context)
    {
        if (@event is TEvent ev)
        {
            await _processFunc.Invoke(ev, context);
        }
    }
}

class RagePassiveProcess : PassiveProcess
{
    public override IEnumerable<IPassiveProcessFunction<PassiveProcess>> FollowingProcesses { get; }

    public RagePassiveProcess()
    {
        FollowingProcesses = new[]
        {
            Create<DamageEvent>(OnAttackedAsync),
        };
    }

    private async Task OnAttackedAsync(DamageEvent @event, BattleContext context)
    {
        Console.WriteLine("Rage");
    }
}
```

これを用いると、 `PassiveProcessHookHandler` をコード生成でまかなう必然性がなくなる。

`IPassiveProcessFunction<TPassive>`, `PassiveProcessFunction<TPassive, TEvent>` は
なるべくライブラリに含めたいが、 `BattleContext` の存在が懸念点。
ここをライブラリに含められないのならば、
`DomainEventHandler` から直接処理できないため、
`PassiveProcessHookHandler` はまだ必要がある。
`BattleContext` を型引数にするという手があるか。

この観点では、攻撃力や防御力などプリミティブ型で表すような値も型として扱うべきだが、
少しやりすぎかもしれない……

## 能力値補正の実装

`Ability` なる能力値のデータ型があったとして、
これをクローンして一部の値を変更したもの返す処理をパッシブ処理に実装するという手がある。
この方法なら、int型として扱うのが自然なものはそう扱える。

この方法は処理のたびに新たなインスタンスを返すので重いかもしれない。
`Ability` が小さなクラスであるほどパフォーマンスへの負荷は小さい。

そこで、 `Ability` の直下を全て参照型のクラスとして扱い、
クローン時に浅いコピーを返すようにすると軽くなりそう。
攻撃力が物理・魔法に分かれているなど二層プロパティを用いる場合にはより自然なやり方である。

変更が何もない場合はクローンせず、元のインスタンスをそのまま返すほうがよい。

この考え方なら、 `Ability` 型をドメインイベントのように入力して
新たな `Ability` を返すものを実装すればライブラリ化できる。

## データストア

データストアをPassiveProcessに注入するコンポーネントのように考えることができるかもしれない？

PassiveProperty が存在して、それはいくつかのコンポーネントを保持する。
PassiveProcessの実装は自分自身にあたる PassveProperty を取得することができて、
コンポーネントから値を取り出すことができる。

PassiveProcessHookHandlerはPassivePropertyに対してイベントを発行する。
PassivePropertyはPassiveProcessに対してメッセージを送るが、このとき自分自身を渡す。
PassiveProcessはPassivePropertyに対して、
Vanishを呼んだりコンポーネントにアクセスしたりできる。

### 2020/11/08

PassiveProcess から派生する StatefulPassiveProcess を実装し、
データストアに型を与えられるようにした。
これで、データストアの型を間違えて使っていれば、コンパイルエラーとなる。

ここまでの設計は以下の通り。

![](out/Passive/Library.png)

### Pure と Stateful の違い

(Pure/Stateful)の区分が(Property,Process,Modifier,ProcessFunction)の区分と直交していて、
このままではクラス数が掛け算で増えてしまう。
Pure性とStateful性という性質を抽象化できないものか？

Property について、

* Pure:
* Stateful: Pureの機能に加えて、データストアを持ち、初期値のロードも行う。

Process について、

* Pure: イベントの登録時にPurePropertyを使うもののみを受付、PureなModifierやFunctionを生成する。
* Stateful: イベントの登録時にStatefulPropertyを使うもののみを受けつけ、StatefulなModifierやFunctionを生成する。

Modifier について、

* Pure: 
* Stateful: Pureと比べると、データストアの型を余分にチェックする。

Modifier について、

* Pure: 
* Stateful: Pureと比べると、データストアの型を余分にチェックする。