## 実はそのUI実装や機能はDIYできちゃう！個人的にも活用している実現Tips集

<p align="right">
<strong>酒井文也 (Fumiya Sakai) Twitter &amp; github: @fumiyasac</strong>
</p>

<hr>

### はじめに

アプリの機能開発時やUI実装時、あるいは基盤となり得る部分に手を入れたりする場合には、自前で準備をする選択肢の他にも、OSSを利用する選択肢も候補に上がる場面も数多くあると思います。また、自前でいざ実装を考えていくとなると __「結構大変そうに思えて二の足を踏んでしまった」__ または __「期間や工数等の制約がある関係でOSSを活用した実装を今回は選択した」__ という経験がある方もいらっしゃるのではないかと思います。

昨今でのiOSアプリ開発では、美しいUI実装や表現を少ない工数で実現する事ができたり、機能開発や設計時の強い味方となるOSSは数多く存在します。そして私自身も平素の業務開発や個人開発時にはその恩恵を大いに享受おり、これらのOSS開発者には感謝と経緯を常に持っています。しかしながら、意図をした仕様や機能を実現する際に、OSS内に搭載されている機能だけでは満たす事が難しい場合や、今後の機能拡張や運用保守等も見据え、 __「OSSで実現している機能や表現等を参考にしながら、現状の仕様に沿った形の内部機能やUI表現を考えてDIYしていく方針」__ を選択する必要がある場面もあるかもしれません。

本稿では、主に __「UIKitまたはSwiftUIを利用した少し難しそうなUI表現に対するDIY実現例」__ 及び __「アプリにおける設計や基盤となり得る部分に対するDIY実現例」__ に対する簡単な事例や実現するためのTipsとなり得る部分を図解等を交えながらご紹介できればと思います。

対象の読者としましては __「一見する難しそうなイメージを持ちやすい部分をDIYする際のヒントや、この様な表現やアプローチを考えていく際の着想を得たいと考えている方」__ 等を想定しております。決して特別な事はありませんが、ほんの少しでも皆様のご参考になる事ができれば嬉しく思います。

### 1. SwiftUIで実現する奥行きがあるCarousel表現

アプリを初回インストールした際のオンボーディング画面や、アプリのTOP画面内の表示Section部分でユーザーの目を引きつける用途で、動きも美しくCarousel表現を利用したい場合があると思います。しかしながら、表示領域内に配置している個々の要素に対して、奥行き表現や重なりを考慮した実装を考えていくと、難しさに加えて表現手段の奥深さがある事例の1つかもしれません。

こちらで紹介する事例は、SwiftUIを利用した画面の配置要素表示のModifierにセットする値に注目し、DragGestureによって変化するX軸方向の表示位置の変化量を利用して、奥行き・重なり・アルファ値・オフセット値等に対応するModifierへ適用したい値を算出する点がポイントになると思います。

![001_draggesture_3d_carousel.png](https://github.com/fumiyasac/iosdc2023_pamphlet_manuscript_vol1/blob/main/images/001_draggesture_3d_carousel.png)

__【🌷Carousel表示領域に適用するModifier処理のコード抜粋】__

```swift
func body(content: Content) -> some View {
    content
        // MEMO: highPriorityGestureを利用してScrollView内で使用しても上下スクロールとの競合を発生しにくくする
        .highPriorityGesture(
            DragGesture(minimumDistance: 20)
            .onChanged({ value in
                // 👉 Carousel要素の移動中はStateと連動するdraggingOffset値を更新する
                draggingOffset = snappedOffset + value.translation.width / 250
            })
            .onEnded({ value in
                // 👉 Carousel要素の移動終了時は自然に元の位置または動かそうとした位置に戻る様にしている
                withAnimation(.linear(duration: 0.16)) {
                    draggingOffset = snappedOffset + value.translation.width / 250
                    draggingOffset = round(draggingOffset).remainder(dividingBy: Double(viewObjectsCount))
                    snappedOffset = draggingOffset
                }
            })
        )
}
```

__【🌷配置要素のModifierにて利用する値を計算するためのメソッド抜粋】__


```swift
// 👉 (補足) campaignBannerCarouselViewObjects: Carousel表示内容を格納する構造体（計算に関与するのは総数のみ）

// MEMO: scaleEffect / opacity / zIndexに必要な調整値を算出する
private func getCarouselElementModifierValueBy(itemId: Int, ratio: CGFloat) -> CGFloat {
    let distanceByItemId = calculateDistanceBy(itemId: itemId)
    return 1.0 - abs(distanceByItemId) * ratio
}

// MEMO: 三角関数(この場合はsinθ)を利用して角度を算出し、奥行きのある重なりを表現する
private func calculateHorizontalOffsetBy(itemId: Int) -> CGFloat {
    let allItemsCount = campaignBannerCarouselViewObjects.count
    let angle = Double.pi * 2 / Double(allItemsCount) * calculateDistanceBy(itemId: itemId)
    return sin(angle) * 200
}

// MEMO: @Stateで定義したdraggingOffsetを利用して間隔値を算出する
private func calculateDistanceBy(itemId: Int) -> CGFloat {
    let allItemsCount = campaignBannerCarouselViewObjects.count
    let draggingOffsetByItemId = (draggingOffset - CGFloat(itemId))
    return draggingOffsetByItemId
        .remainder(dividingBy: CGFloat(allItemsCount))
}
```

__【🍊その他Carousel表現やGrid表現をするための事例紹介】__

- __SwiftUIで作る「Drag処理を利用したCarousel型UI」と「Pinterest風GridレイアウトUI」の実装例とポイントまとめ__
  - 記事URL: https://qiita.com/fumiyasac@github/items/b5b313d9807ff858a73c

### 2. UIKitを利用した複数の要素が絡む様な表現例

無限循環型のタブ表示を伴った多くの記事や商品写真等を整理して配置する画面表現処理や、シームレスで美しい動きを伴う画面遷移処理については、要望受けたり憧れて自分で試してみた経験がある方もいらっしゃるかもしれません。

こちらで紹介する事例は、実装の兼ね合いによっては条件や制約がつく場合もありますが、UICollectionView・UIScrollView・UIPageViewController・CustomTransitionを応用して組み合わせた処理におけるアイデアとポイントを図解で整理したものになります。

![002_infinite_scroll_tab_contents.png](https://github.com/fumiyasac/iosdc2023_pamphlet_manuscript_vol1/blob/main/images/002_infinite_scroll_tab_contents.png)

__【🍊紹介した方針のタブ型無限循環スクロールUI実現時の解説記事紹介】__

- __ライブラリなしでメディアアプリでよく見る無限スクロールするタブの動きを実装したUIサンプルの紹介__
  - 記事URL: https://qiita.com/fumiyasac@github/items/af4fed8ea4d0b94e6bc4

__【🍊画面上部を引っ張って閉じる処理と画面遷移トランジションの組み合わせに関する図解】__

![003_drag_to_dusmiss_transition.png](https://github.com/fumiyasac/iosdc2023_pamphlet_manuscript_vol1/blob/main/images/003_drag_to_dusmiss_transition.png)

### 3. SwiftUIベースのProjectにおけるRedux処理機構

PropertyWrapperを活用した事例として、Redux処理機構における事例を簡単に紹介します。Redux自体の詳細な解説は本稿では割愛しますが、単一方向でのデータフローを実現する＆散らばりがちな状態を一元管理が可能な点等に大きな特徴があります。

Redux処理における一番のおおもとになる部分はStoreクラスとなり、アプリ全体のState(各画面用Stateを集約したもの)を __「@Published (変数定義)」__ の形で定義し、各画面用Stateに変化があった際には伴ってView要素の状態も変化する形を取る様にしています。そして各画面用Stateの状態更新を実行するためのActionを発行する __「関数: func dispatch(action: Action)」__ を定義しています。

Reduxを利用した画面更新処理における概要図と全体処理フロー及び、Storeクラス定義をまとめると下記の様な形となります。

![004_redux_diagram_and_flow.png](https://github.com/fumiyasac/iosdc2023_pamphlet_manuscript_vol1/blob/main/images/004_redux_diagram_and_flow.png)

__【🌷Storeクラス内全体の概要＆説明】__

```swift
final class Store<StoreState: ReduxState>: ObservableObject {

    // MARK: - Property

    // 👉 MEMO: @Publishedで定義し、各画面用Stateに変更があった場合にView側の表示を変更する。
    // (1) 各画面に対応するStateは「例. AppState」の様な名前のStruct内で一元管理をされている。
    // (2) アクセス修飾子は`private(set)`としているので各画面用Stateの内容を直接書き換えることはできません。
    @Published private(set) var state: StoreState
    private var reducer: Reducer<StoreState>
    private var middlewares: [Middleware<StoreState>]

    // MARK: - Initialzer

    init(
        reducer: @escaping Reducer<StoreState>,
        state: StoreState,
        middlewares: [Middleware<StoreState>] = []
    ) {
        self.reducer = reducer
        self.state = state
        self.middlewares = middlewares
    }

    // MARK: - Function

    // 👉 Actionを発行するDispatcher関数定義
    func dispatch(action: Action) {
        // MEMO: 任意のActionに対応するReducer関数を実行して対応するState内容を更新する。
        // (1) 各画面に対応するReducerは純粋関数で定義し、任意のAction関数から受け取った値を元に各画面に対応するState内容を更新する。
        // (2) 各画面に対応するReducerはStateと同様に「例. AppReducer」の様な名前の関数内で一元管理をされている。
        Task { @MainActor in
            self.state = reducer(
                self.state,
                action
            )
        }
        // MEMO: 任意のAction発行後に「API非同期通信処理・データ永続化処理」実行するために、各Actionに対応するMiddleware関数を実行する。
        middlewares.forEach { middleware in
            middleware(state, action, dispatch)
        }
    }
}
```

__【🍊実際にこのStoreクラスを利用したRedux機構を利用した事例紹介】__

簡単ではありますが、実際にUI実装や簡単なAPI非同期通信処理・データ永続化処理を含んだ、本稿で紹介しているRedux機構を活用したサンプル実装について詳細に解説した記事も別途用意しております。

- __SwiftUI+Reduxを利用したUI実装サンプルにおけるポイント解説__
  - 記事URL: https://zenn.dev/fumiyasac/articles/01f1bc86bf8c40
  - GitHub: https://github.com/fumiyasac/SwiftUIAndReduxExample

### まとめ

本稿でご紹介した事例は、これまで私自身が個人的に取り組んできたものや、業務内のコードで実際に活用をするためにプロトタイプとして作成したもの・一部改善等をしてプロダクションコードとして適用した事例における一部をご紹介しましたが、これらの表現や実装は私自身も取り組んでいた時に思わず __「なるほど！この様な方法があったか！」__ という驚きを感じたものをピックアップしてみました。

今回紹介したTips集の中で、複雑なUI表現する際の実装アイデアやポイントの事例については、実装以上に __「画面全体のView要素内において対象の要素がどの様な位置関係にあるか？」__ という点への配慮を要する場合がある点に難しさを感じる事もあると思いますし、Redux処理機構のおける部分についてはPropertyWrapperの性質を活用した事例になるかと思います。

一見すると難解で自分では手が出そうにないのでは？と思える様な実装についても、実装や構成の工夫を加えたり、要点や概略を整理していく事でヒントや方針が見える場合もありますので、自分にとっての __「憧れの機能を実現している事例や知っているアプリの気になる実装に注目」__ しながら設計や実装を深掘りをしていくと、周辺知識の理解とも相まって解像度がより高まっていくのではないかと考えております。

最後までお読み頂きまして本当にありがとうございました。
（※紙面の関係で概要紹介のみとなってしまい本当にすみませんでした...😓）
