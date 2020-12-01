
### BattleFlow
バトルの流れを記述する。
バトル終了処理を担う。

### BattleContext

* バトル全体に渡って存在する全ての情報を保持する。
* ターゲット可能なものを列挙することができる。Battlerの他にも、フィールドやアイテムなどをターゲットにするゲームも想定。

### IOrderDetermination

Battlerの行動順を決定する。

### Battler : ITargetable

* BattleContextと自分を合わせてITurnContextを作れる。
* ITurnContextの作り方は**ユーザー定義**
* **追加の**属性を使えるようにすべき？

例
* 継承で独自のBattlerを作成する。
	* このBattlerは敵と味方を区別でき、与えられたターゲットから敵をフィルターしたり、味方をフィルターしたりできる。

### ActiveAttribute

* ActiveEffectを持つ。
* TargetRangeを持ち、ターゲティング時に参照する。
* **追加の**属性を使えるようにすべき？

### TargetRange

* TargetFilterを持つ。
* ターゲット選択を実際に行う。
	* その際 TargetingStrategy に選択操作を委譲できる。
	* 選択処理は**ユーザー定義**
* **追加の**属性を使えるようにすべき？

### TargetFilter

* 渡されたBattlerから、選択可能なBattlerを絞り込む。
* 絞り込み方は**ユーザー定義**

### ActiveEffect

* アクティブ効果を実行する。
* イベントの発行に至るまでの処理は**ユーザー定義**
* **追加の**属性を使えるが、ゲーム中変化しないこと。
* BattleEventHandlerを受け取り、イベントを発生させることができる。

### PassiveAttribute

* PassiveEffectを持つ。

### PassiveAttributeManager

* EventHandlerをフックして、実行中のPassiveEffectの自発的な削除ができるようにする。

### PassiveEffect

* パッシブ効果を実行する。
* イベントの発行に至るまでの処理は**ユーザー定義**
* **追加の**属性を使えるが、ゲーム中変化しないこと。
* BattleEventHandlerを受け取り、イベントを発生させることができる。

### BattleEventHandler&lt;TBattleEvent&gt;

* ActiveEffect/PassiveEffect からイベントを受け取るレシーバー。
* イベントを受け取った際の処理は**ユーザー定義**
* どのようなイベントを受け取ることができるのかは**ユーザー定義**
	* ジェネリクスでインターフェースなどを指定してよい。

### TargetingStrategy

* ターゲット選択において、実際の選択操作を受け付ける。
* TargetRangeにて選択可能な相手や人数が決まったら、その情報を元に呼ばれる。
* 選択操作がどのようなものなのかは**ユーザー定義**

### TurnContext

* BattleContext + Battler

### TargetingContext

* TurnContext + ActiveAttribute
* ここに至るまでの、メイン作戦やスキル選択は**ユーザー定義**
* ターゲット選択に移るので、TargetRangeのメソッドが呼び出される。

### InvocationContext

* TargetingContext + ActiveTarget
* ActiveAttributeの効果を実際に実行する。

### ActiveTarget

* ターゲットとなるものが多岐にわたる可能性があるため抽象化。
* TargetUnitの配列を持つ。

### TargetUnit

* あるアクティブ効果の、特定の効果を同様に受けるグループ。
* スキルなどが複数の効果を持つ場合、ある効果はグループA、次の効果はグループBにだけ、という設定ができる。
* ITargetable の配列を持つ。

### Ability

* Battlerの性能を表す。バトル中においてイミュータブル。
* **追加の**属性を持てる。

## パッシブ効果をインターフェースで？
IPassiveEffectOnAttack なるインターフェースで、攻撃時に発動する効果を定義する。このようにすると、この実装クラスは死亡時のパッシブ効果などのメソッドを持たずに済む。

PassiveAttributeManagerは (インターフェースType, List&lt;PassiveAttribute&gt;) のディクショナリを持つようにする。実際の呼び出しは(Type, Context)を引数にRunなるメソッドを呼べばよい。

Contextをどのように自由にするかが課題？

```csharp
public static Task On$timing$(this IBattler battler, $context_type$ context)
{
	return battler.StatusManager.RunAsync<$passive_interface_type$>(effect => effect.RunAsync(context));
}
```

## 選んだ敵が実際には居なかった場合の処理は？

## ジェネリクスを使わずに、ユーザー定義のBattlerなどを違和感なく使いたい

BattleContext.Players などは IEnumerable&lt;Battler&gt; 型である。
ユーザーはダウンキャストして使わなければならなくなる。
ダウンキャストなしで使うことはできないだろうか？
ジェネリクスは至る所で利用するには煩雑すぎる……

もしアイテムをスキルの対象にできるようになったなら、アイテムの型までジェネリクスで指定するはめになってしまう……

ContextクラスにBattlerなどを持ってしまっているのが問題。
Duptip.Battle内で使うための Playersゲッターと、ユーザーが使うためのPlayersゲッターは分けるべき。
例えば、BattleContextクラスとUserContextクラスで分ける。
こうしたとしても、UserContextの型をジェネリクスで明示しなければならないのは変わらない。
その代わり、 Invocation&lt;Battler, Item, Skill, Field, ...> のようにしなければものは、 Invocation&lt;TContext> で済む。

ActiveEffect などで基底クラスに毎回コンテキストの型を明示しなければならないが、中間にクッションとなるクラスに継承させれば緩和できる。

Battlerを裸で渡される部分は相変わらず扱いづらいままになる……