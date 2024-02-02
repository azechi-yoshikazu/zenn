---
title: "[UE5]エディタ右下のツールバーに独自アイテムを追加する[エディタ拡張]"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["UnrealEngine", "cpp"]
published: false
---

## はじめに

C++での内容となります。
Blueprint でも可能ですが実用性のある記事と出来そうになかったため断念……

### 何をするのか

エディタ下部にあるステータスバーの右側
ここにあるツールバーに対して独自のアイテムを追加したい
![](/images/ue-toolbar-extension/editor-before.png)
_矢印の先に出したい_

### 環境

- Unreal Engine 5.3.2
- Visual Studio 2022

## 本題

### 1. エディタ拡張用のモジュール追加

ここではプラグインとして用意していきます。
プラグインではなくモジュール追加でも問題ありませんが、手順としてわかりやすいためプラグインとしています。

#### 1-1. プラグインの追加

`Editor > Plugins`から Plugins ウィンドウを開き、Add ボタンから新規の Blank プラグインを追加します。
今回は`ToolBarExtension`という名前にしました。

![](/images/ue-toolbar-extension/new-plugin.png)
_新規プラグイン作成ウィンドウ_

#### 1-2. プラグインのモジュール設定を変更

Blank で作成したプラグインのモジュールはランタイム用で作成されます。
今回はエディタ用に用意するので、モジュール設定をエディタ用に変更しましょう。

```cpp:ToolBarExtension.uplugin
"Modules": [
    {
        "Name": "ToolBarExtension",
        "Type": "Editor",// <--Runtimeから変更
        "LoadingPhase": "Default"
    }
]
```

### 2. 表示するための Widget を作成

ツールバーに表示する Widget を用意します。
ここでは説明のためボタンのみのシンプルなものにして、モジュールのスタートアップのタイミングで生成しておきます。
この手順ではまだ表示するための実装は行わないためエディタの見た目に変化はありません。

```cpp:ToolBarExtension.cpp
#include "Widgets/Input/SButton.h"

...

void FToolBarExtensionModule::StartupModule()
{
    auto Widget =
        SNew(SButton)
        .Text(LOCTEXT("ToolBarExtension_Button_Title", "Button"))
        // クリックしたらログが出る（Warningなのは色がついて目視しやすいため）
        .OnClicked_Lambda([]()
            {
                UE_LOG(LogTemp, Warning, TEXT("MyTool Button Clicked!!"));
                return FReply::Handled();
            });
}
```

### 3. ツールバーに用意した Widget を追加

前の手順で作成した Widget をツールバーに追加してエディタから操作できるようにしていきます。

#### 3-1. 必要なモジュール依存設定を追加

```cs:ToolBarExtension.Build.cs
PrivateDependencyModuleNames.AddRange(
    new string[]
    {
        "CoreUObject",
        "Engine",
        "Slate",
        "SlateCore",
        "ToolMenus",// <--追加
    }
    );
```

#### 3-2. ツールバーへの項目追加を実装

```cpp:ToolBarExtension.cpp
#include "ToolMenus.h"

...

void FToolBarExtensionModule::StartupModule()
{
    // 追加するツールバーの取得
    if(UToolMenu* Menu = UToolMenus::Get()->ExtendMenu(TEXT("LevelEditor.StatusBar.ToolBar")))
    {
        auto Widget = ...; // 追加できそうなタイミングのみ生成したいので移動

        // 取得したツールバーに対してセクションを追加
        FToolMenuSection& MyToolSection = Menu->AddSection(TEXT("MyTool"), FText::GetEmpty(), FToolMenuInsert(NAME_None, EToolMenuInsertType::First));

        // 追加したセクションの項目として、作成しておいた Widget を登録
        MyToolSection.AddEntry(
            FToolMenuEntry::InitWidget(TEXT("MyToolBar"), Widget, FText::GetEmpty(), true, false)
        );
    }
}
```

これでボタンが出ました！

![](/images/ue-toolbar-extension/editor-after.png)
_出た！……けど Trace の右側……？_

#### 3-3. 一番左に出したい……

なんとなく、拡張したセクションは左にすべてまとまっていてほしいです。
`UToolMenu::AddSection()`に渡している`FToolMenuInsert`は、追加済みのセクションを基準とした挿入が行えるようなので試してみます。
今回追加しようとしているタイミングよりも早く追加されているセクションがある場合、そのセクションを指定します。
追加済みセクションよりも前（左側）に出てくれることを期待して、`EToolMenuInsertType::Before`を指定してみます。

```cpp:ToolBarExtension.cpp
void FToolBarExtensionModule::StartupModule()
{
    // 追加するツールバーの取得
    if(UToolMenu* Menu = UToolMenus::Get()->ExtendMenu(TEXT("LevelEditor.StatusBar.ToolBar")))
    {
        ...

        // 取得したツールバーに対してセクションを追加
        const FToolMenuInsert Position(Menu->Sections.Num() > 0 ? Menu->Sections[0].Name : NAME_None, EToolMenuInsertType::Before);// <-- 追加
        FToolMenuSection& MyToolSection = Menu->AddSection(TEXT("MyTool"), FText::GetEmpty(), Position);

        ...
    }
}
```

![](/images/ue-toolbar-extension/editor-final.png)
_望んでいた配置になった_

うまくいきました！
ただし、今回のやり方はたまたま上手くいっただけに近いです。
今後のエンジン更新などで処理順が変わってしまう可能性があるため、モジュールの`LoadingPhase`の変更などが必要かなと思います。

## まとめ

思っているよりも簡単にツールバーへの追加は出来ました。

今回使用している UToolMenu は Blueprint にも公開されているため、ちゃんと調べれば Editor Utility Object などから追加することも可能だと思います。

Slate をベースとしているため少しとっつきにくいところはあるかもしれませんが、
バージョン管理のハッシュ値を出しておくだけでもチーム開発では便利だったりするので是非試してみてください。
