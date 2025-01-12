# 副作用とプロシージャ

これまで`print!`の`!`の意味を説明せずにいましたが、いよいよその意味が明かされます。この!は、ズバリこのオブジェクトが「副作用」のある「プロシージャ」であることを示しています。プロシージャは関数に「副作用」という効果を与えたものです。

```erg
f x = print! x # EffectError: functions cannot be assigned objects with side effects
# hint: change the name to 'f!'
```

上のコードはコンパイルエラーになります。関数中でプロシージャを使用しているからです。このような場合は、プロシージャとして定義しなくてはなりません。

```erg
p! x = print! x
```

`p!`, `q!`, ...は、プロシージャを表すための典型的な変数名です。
このようにして定義したプロシージャもまた関数内では使用できないため、副作用は完全に隔離されるわけです。

## メソッド

関数とプロシージャにはそれぞれメソッドが存在します。関数メソッドは`self`の不変参照のみを取れ、プロシージャルメソッドは`self`の可変参照を取れます。
`self`は特殊な引数で、メソッドの文脈では呼び出したオブジェクト自身を指します。参照の`self`は他のいかなる変数にも代入できません。

```erg
C.
    method ref self =
        x = self # OwnershipError: cannot move out 'self'
        x
```

メソッドは`self`の所有権を奪うこともできます。そのメソッドの定義では`ref`または`ref!`を外します。

```erg
n = 1
s = n.into(Str) # '1'
n # ValueError: n was moved by .into (line 2)
```

可変参照を持てるのは常に1つのプロシージャルメソッドのみです。さらに可変参照が取られている間は元のオブジェクトから参照を取れなくなります。その意味で`ref!`は`self`に副作用を引き起こします。

ただし、可変参照から(不変/可変)参照の生成はできることに注意してください。これによって、プロシージャルメソッド中で再帰したり`self`を`print!`できたりします。

```erg
T -> T # OK (move)
T -> Ref T # OK
T => Ref! T # OK (only once)
Ref T -> T # NG
Ref T -> Ref T # OK
Ref T => Ref! T # NG
Ref! T -> T # NG
Ref! T -> Ref T # OK
Ref! T => Ref! T # OK
```

## Appendix: 副作用の厳密な定義

コードに副作用があるかないかのルールはすぐに理解できるものではありません。
理解できるようになるまでは、とりあえず関数として定義してエラーが出ればプロシージャとするコンパイラ任せのスタイルを推奨します。
しかし、言語の厳密な仕様を把握しておきたい人のために、以下ではもう少し詳しく副作用について説明します。

まず、Ergにおける副作用に関して、戻り値の同値性は関係がないということに注意してください。
任意の`x`に対して`p!(x) == p!(x)`となるプロシージャが存在します(常に`None`を返すなど)し、`f(x) != f(x)`となる関数も存在します。

前者の例は`print!`で、後者の例は以下の関数です。

```erg
nan _ = Float.NaN
assert nan(1) != nan(1)
```

また、クラスや関数のように同値判定自体ができないオブジェクトも存在します。

```erg
T = Structural {i = Int}
U = Structural {i = Int}
assert T == U

C = Class {i = Int}
D = Class {i = Int}
assert C == D # TypeError: cannot compare classes
```

本題に戻ります。Ergにおける「副作用」の正確な定義は、

* 外部の可変な情報にアクセスすること

です。外部とは、一般には外側のスコープを指します。Ergがタッチできないコンピューターリソースや、実行前/後の情報については「外部」に含まれません。「アクセス」は書き込みだけでなく読み込みも含めます。

例として`print!`プロシージャについて考えます。`print!`は一見何の変数も書き換えていないように見えます。しかし、もしこれが関数だったとすると、例えばこのようなコードで外側変数を書き換えられます。

```erg
camera = import "some_camera_module"
ocr = import "some_ocr_module"

n = 0
_ =
    f x = print x # 仮にprintを関数として使えたとします
    f(3.141592)
cam = camera.new() # カメラはPCのディスプレイを向いています
image = cam.shot!()
n = ocr.read_num(image) # n = 3.141592
```

`camera`モジュールはあるカメラ製品のAPIを提供する外部のライブラリ、`ocr`はOCR(光学文字認識)のためのライブラリとします。
直接の副作用は`cam.shot!()`によって引き起こされていますが、明らかにその情報は`f`から漏洩しています。よって、`print!`は性質上関数とはなれません。

とはいえ、関数中で値を一時的に確認するとき、そのためだけに関連する関数まで`!`を付けたくない場合もあるでしょう。その際は`log`関数が使えます。
`log`はコード全体の実行後に値を表示します。これにより、副作用は伝搬されません。

```erg
log "this will be printed after execution"
print! "this will be printed immediately"
# this will be printed immediately
# this will be printed after execution
```

つまり、プログラムへのフィードバックがない、言い換えればいかなる外部オブジェクトもその情報を使うことが出来ないならば、情報の「漏洩」自体は許される場合があります。「伝搬」されなければよいのです。

<p align='center'>
    <a href='./06_operator.md'>Previous</a> | <a href='./08_procedure.md'>Next</a>
</p>
