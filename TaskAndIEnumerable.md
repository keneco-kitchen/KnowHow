##IEnumerableについて
#IEnumerableインターフェース
みなさん。foreach使っていますか！？便利ですよね。
foreachが使える条件は、Listクラスのように対象がIEnumerableインターフェースを実装していることです。
IEnumerableインターフェースは下記の通り
```cs
public interface IEnumerable
{
  IEnumerator GetEnumerator();
}
```
IEnumeratorって何よ、というと
```cs
public interface IEnumerator
{
  object Current { get; } // 現在指している要素
  bool MoveNext();        // 次の要素があるか？
  void Reset();           // 指し先を最初に戻す
}
```
#foreachの中身
GetEnumeratorを呼んで、IEnumeratorを取得します。IEnumeratorは方向指示器のようなものです。
取得した方向指示器から、
1. 今の要素**Current**を取得し、
2. 次の要素があるかを確認**MoveNext**を呼び、falseなら終了し、
3. trueなら1の処理に戻ります

イメージはこんな感じ
↓Current
■■■

MoveNext:true

  ↓Current
■■■

MoveNext:true

    ↓Current
■■■

MoveNext:false

#イテレータブロック(yield return)
foreachを使えるようにするには、上記のインターフェースの実装が必要ですが、C#2.0以降は、
イテレータブロックという機能を使用すれば、インターフェースの実装は必要なくなりました。

例えば、下記の処理
```cs
void hoge()
{
  var l_members = new List<string>()
  {
    "さくま",
    "ひらおか",
    "いとう",
    "すずき",
    "しんじくんは欠席",
    "まぁまぁ",
  };

  foreach(var l_member in l_members)
  {
    Console.WriteLine(l_member);
  }
}
```
と同等の動きが、下記で実現できます
```cs
void hoge()
{
  foreach(var l_member in GetMember())
  {
    Console.WriteLine(l_member);
  }
}

IEnumerable<string> GetMember()
{
  yield return "さくま"
  yield return "ひらおか"
  yield return "いとう",
  yield return "すずき",
  yield return "しんじくんは欠席",
  yield return "まぁまぁ",
}
```
つまり、**yield return**に到達すると、そこで処理が返り、次回のforeachは前回位置から開始します。

##Taskについて
#async,await
以下のような処理を書くと、UIがフリーズします。
なぜならば、メインスレッド(UIスレッド)がTreadSleepによって、待機してしまいます。
UIを更新できるのはメインスレッドだけなので、結果としてUIがフリーズしてしまいます。
つまり、重たい処理は別のスレッドに処理を任せ(非同期処理)、メインスレッドを楽にさせる必要があります。
.Netのバージョンが上がるにつれて非同期処理の変遷があったようで、現在はTaskクラスを使うのが一般的だそうです。
Taskを使用すると、Threadの使用を隠蔽し、直感的に記述できます。

Task.Run(()=>Thread.Sleep)

なんて書くことにより、重たい処理を別スレッドに処理させることができます。
ただ、上記のままだと、Task完了前に処理が終了してしまう問題があります。
1. 重たい処理は別スレッドに任せたい、
2. しかしその下に続く処理はTask完了後処理(継続タスクという)をしたい、
3. ただし、スレッドをTask完了するまで止めておきたくはない。
これを実現できるのがawaitです

await Task.Run(()=>Thread.Sleep)

と記述すると、awaitまでスレッドが進むとTaskが完了するまでスレッドが開放され、awaitの処理以降は継続タスク
とみなします。Taskが完了すると継続タスク別のスレッドによって開始します。
ただし、awaitしたスレッドがメインスレッドであった場合は継続タスクはメインスレッドによって処理が再開されます。
awaitするためには、メソッドにasyncを付ける必要があります。

#キャンセルトークン
実行中のタスクを途中でキャンセルしたい場合、安全に停止する方法の１つとしてCancellationTokenを使う方法があります。
メインスレッドでキャンセルフラグを立てると実行中タスクのトークンに伝わります。
タスク側はフラグが立っていないかをポーリングすることによって、キャンセルを監視することができます。

#IProgress<T>インターフェース
タスクの処理中に画面を更新したいことがあります。
そんなときに、タスク処理中にメインスレッドをするためのコールバックの仕組みがあります。

##タスクとイテレータブロックを使い進捗を確認する
キャンセルフログをチェック、進捗を報告する、という処理を定期的に行いたいため、
非同期処理をループ文で書きたくなります。
ただ、非同期処理がシーケンシャルなとき、少々骨が折れます。
例えば、シーケンスを小分けにし、小分けにした処理ごとに共通のインターフェースで実装し、
リスト化することによって、ループ処理として扱えるようになります。

そこで、イテレータブロックを使い、下記のようにすることで、foreachを使用できます。
void DoWork()
{
  // シーケンスの順次実行
  foreach(var l_proc in Secuence())
  {
    // キャンセルフラグをチェック
    if(l_cancel.IsCancelRequire){ return; }
    // 処理済みカウントアップ
    l_doneCount++;
    // 画面更新依頼
    l_reporter.Report(p_progress);
  }
}

IEnumerable<StatusEnum> Sequence()
{
  void ProcA();
  yield return StatusEnum.OK;
  void ProcB();
  yield return StatusEnum.OK;
  void ProcC();
  yield return StatusEnum.OK;
}
