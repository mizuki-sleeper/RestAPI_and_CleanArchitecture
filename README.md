# REST and Clean Architecture

Golang の実装を通して、REST API とクリーンアーキテクチャについて学んだ備忘録です。

コードは簡単な ToDo アプリの、バックエンドのみの実装です。
<br>

# REST API

REST API とは設計思想の事。

2000 年に Roy Fielding という方が、以下の REST と呼ばれる設計原則を提唱したらしい。これを Web API に適用したのが、REST API。
<br>

### REST の 4 原則

1. セッションなどの状態管理を行わず、やり取りされる情報はそれ自体で完結して解釈することができる
2. 情報を操作する命令の体系が予め定義・共有されている
3. すべての情報は汎用的な構文で一意に識別される
4. 情報の一部として、別の状態や別の情報への参照を含めることができる

以下、自分が重要だと思ったことをまとめる。
<br>

## アドレス可能性と統一インターフェース

REST API は HTTP 技術を元にしていて、リソースは一意の URI で表し、リソースに対する操作はメソッドで表現する。

例えば、GET メソッドはリソースの取得に使用され、POST メソッドは新しいリソースの作成に使用する。

リソースを削除するときに、PUT メソッドなどが使えなくもないが、データを削除する時は DELETE メソッドを使うというように、統一させる。

そのため、URL を見ただけでは、どんな操作がわからないとも言える。

```
https://example.com/api/products/{product_id}
```

上記のような一位のリソースを表す URI に対し、HTTP のメソッドを通じて、操作を決定する。

```
https://example.com/api/createuser
https://example.com/api/deleteuser

https://example.com/api/products
https://example.com/api/items

https://example.com/api/users/1
https://example.com/api/users?id=1
```

上記のように、URI に動詞を入れたり、命名規則やパラメータの使用に一貫性がない設計は REST API の思想からは外れる。

そしてレスポンスのステータスコードを適切なものに設定したり、データのフォーマット（現在は JSON が多い）を統一させることも重要。
<br>

## ステートレス性

エンジニア界隈で有名な「HTTP はステートレスなプロトコルである。」という言葉にある HTTP のステートレス性を活用する。

状態を保存しないことで、管理を容易にし、負荷にも強くなる。
<br>

## まとめ

共通の枠組みを作ることで、使用者、開発者共に理解しやすいものとなる。

また、ステートレスなためスケーラビリティが向上する。

エンジニアらしい、思想だと感じた。
<br>

# クリーンアーキテクチャ

クリーンアーキテクチャはよく有名な下の図のように説明される。

<img src="https://www.kabuku.co.jp/wp/wp-content/uploads/2019/06/CleanArchitecture.jpg">

簡単にいうと、「疎結合にすることで変更用意性を確保しよう」、という話なのかな〜と理解している。

色々調べる中で、「一番重要な概念（絶対必要な条件）を中心にして、そこに向かって依存するような設計にする」。これが一番重要なのかなと思う。
<br>

## 前提

前提として、クリーンアーキテクチャについてはさまざまな見解があり、先に示した図のように４層からなるアーキテクチャを指す場合や、依存関係について着目し、図のような 4 層に拘らない考え方もあります。

そこで、まずは前者の構成で考え、その後後者の意見も踏まえ設計全般について考えていこうと思います。

調べていく中で、クリーンアーキテクチャと呼ぶかはわからないけど、アーキテクチャの設計はこうした方がいいよなーと考えるきっかけになったので、それについても話そうと思います。

## クリーンアーキテクチャ概要

先の図を元に ToDo アプリを例に考えます。

### Enterprise Business Rules(Entities)

縁の中心 Entities の部分は、アプリのビジネスロジックやドメインのルールを表現します。ToDo アプリで言えば、タスクやユーザーはルールとして必要になります。

```
// 例
type Task struct {
	ID        uint      `json:"id" gorm:"primaryKey"`
	Title     string    `json:"title" gorm:"not null"`
	CreatedAt time.Time `json:"created_at"`
	UpdatedAt time.Time `json:"updated_at"`
	User      User      `json:"user" gorm:"foreignKey:UserId; constraint:OnDelete:CASCADE"`
	UserId    uint      `json:"user_id" gorm:"not null"`
}
```

このようにデータがどのような構造を持つかなどを定義します。

### Application Business Rules(Use Cases)

赤色の層、usecases では、ToDo アプリの主要な操作や機能を定義します。こちらでは、システムとしてのルールを定義します。フォームの値が正常であればタスクを保存するなどです。

```
// 例
func (tu *taskUsecase) GetAllTasks(userId uint) ([]model.TaskResponse, error) {
	tasks := []model.Task{}
	if err := tu.tr.GetAllTasks(&tasks, userId); err != nil {
		return nil, err
	}
	resTasks := []model.TaskResponse{}
	for _, v := range tasks {
		t := model.TaskResponse{
			ID:        v.ID,
			Title:     v.Title,
			CreatedAt: v.CreatedAt,
			UpdatedAt: v.UpdatedAt,
		}
		resTasks = append(resTasks, t)
	}
	return resTasks, nil
}
```

### Interface Adpters(Controllers)

このレポジトリの ToDo アプリの場合、緑色の層の Controller はユーザーからのリクエストを受け取り、適切な処理を行い、レスポンスを返す役割を担います。

```
// 例
func (tc *taskController) GetAllTasks(c echo.Context) error {
	user := c.Get("user").(*jwt.Token)
	claims := user.Claims.(jwt.MapClaims)
	userId := claims["user_id"]

	tasksRes, err := tc.tu.GetAllTasks(uint(userId.(float64)))
	if err != nil {
		return c.JSON(http.StatusInternalServerError, err.Error())
	}
	return c.JSON(http.StatusOK, tasksRes)
}
```

### Frameworks & Drivers

最後に、青色の層（今回の Todo アプリにおける router）では、クライアントからのリクエストを受け取り、対応するコントローラーにルーティングするなどの役割を果たします。また DB やフレームワークに依存するコードもここに書きます。

```
// 例
func NewRouter(uc controller.IUserController, tc controller.ITaskController) *echo.Echo {
    中略
	t.GET("", tc.GetAllTasks)
	t.GET("/:taskId", tc.GetTaskById)
	t.POST("", tc.CreateTask)
	t.PUT("/:taskId", tc.UpdateTask)
	t.DELETE("/:taskId", tc.DeleteTask)
	return e
}
```

このようにした場合依存関係を、円の外側から中心に向かって依存するようにします。そうすることで、外側の層で変更があっても、内側の層には影響を与えない。逆に Entities に変更があった場合は全ての層でそれに合わせて変更をしないといけないことになります。

たとえば、usecases では、タスクを保存するなどの関数が定義され、これはベータベースを呼ばなければいけないので、このままでは依存関係は外側に向いています。これを避けるために例えば、usecases の層にインターフェースを実装し、それぞれをインターフェースに依存させます。そうすることで usecases からみたらインターフェースは同じ層にあり、外側のデータベースのコードから見たら、インターフェースはより内側の層にあるので、全体で見たときに、依存関係が内側に向くことになります。これがいわゆる依存性逆転の法則というものです。

クリーンアーキテクチャではこのようにして、内側に向かって依存関係が向かっていくように設計します。

一意な定義は見つかりませんでしたが、以下のような原則があるようで、これを見るとなんとなくしたいことは分かるのではないかと思います。

- 依存関係逆転の原則（Dependency Inversion Principle）
- 依存関係の反転（Dependency Inversion）
- レイヤードアーキテクチャ（Layered Architecture）
- ドメイン駆動設計（Domain-Driven Design）の原則
- 単一責任の原則（Single Responsibility Principle）

クリーンアーキテクチャは、ソフトウェアの各コンポーネントやレイヤーの責任と依存関係を明確化し、変更に対して柔軟なシステムを構築することを目指す設計思想のようです。

メリットとして、ビジネスロジックと技術的な詳細を分離することで、システム全体の保守性や拡張性を高めることができます。

## よりよい設計とは

ここまでクリーンアーキテクチャについて考えてみて、必ずしも最初に示した図のような設計にしなくてもよいという意見もあるわけなのですが、この意見を汲むとすると、クリーンアーキテクチャの本質的な部分は、外から内側に依存関係を向けること(ただしレイヤーをどのように配置してもよい(必ずしも一番内側が Enterprise Business Rules である必要はない))なのだと思います。

ではどういう設計にするのかという話なのですが、ここで、一般論を考えてみます。

Web アプリを開発する時は、一般的に UI の部分は変更が多くなると思います。この UI の部分にアプリの大部分が依存していたらどうでしょうか。当然変更が多くなります。

このような設計はいけないわけで、逆に言えば、変更が少ない部分をあの円の図でいうところの中心に配置すればどうでしょうか。自分が作りたいアプリで絶対に必要な機能はどれなのか、MVP なども考えて、変更可能性の低いものから中心になるように配置し、依存性を向けていけば良いと思うのです。これが私が初めに言った「一番重要な概念（絶対必要な条件）を中心にして、そこに向かって依存するような設計にする」という言葉の意味です。

## まとめ

まとめると、クリーンアーキテクチは、ソフトウェアシステムを構築する際の設計原則とパターンの集合体です。ソフトウェアの保守性、テスト容易性、柔軟性、拡張性を向上させることを目指しています。

クリーンアーキテクチャを調べていくと難しい用語がたくさん出てきて、ここにまとめた以上に細かい思想が述べられています。そのため私自身完全に理解できたわけでなくできる限りでまとめてみましたが、システムの構成要素を分離し、柔軟性や保守性を向上させるためにはどうすればよいのかを考えるよいきっかけになりました。
<br><br><br>

P.S. わかりづらかったらすみません...備忘録ということで許してください 🙇‍♀️
