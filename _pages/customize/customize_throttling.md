---
layout: single
title: スロットリング機能のカスタマイズ
keywords: core カスタマイズ スロットリング
tags: [core, throttling]
permalink: customize_throttling
folder: customize
---


---

## スロットリング機能

※この機能はEC-CUBE 4.2.1から利用できます。

[EC-CUBE/ec-cube#5881](https://github.com/EC-CUBE/ec-cube/pull/5881)

クレジットマスター等への抑止として、スロットリング機能が追加されました。

IPごと、または会員ごとのスロットリングを行うことができます。

本体で適用される対象機能や試行回数等は以下となります。

| 画面 | ルーティング | IPベース | 会員ベース | 備考 |
|---|---|---|---|---|
| 会員登録 | entry | 5回/30分 | - |  |
| パスワード再発行 | forgot | 5回/30分 | - |  |
| 問い合わせ | contact | 5回/30分 | - |  |
| 注文確認画面 | shopping_confirm | 25回/30分 | 10回/30分 | バリデーション等が完了し、決済プラグインへ処理を以上する直前で実施 |

また、プラグインやCustomizeからもこの機能を利用できます。

## 拡張サンプル

### yamlでの設定

以下のようにyamlで設定を行うことで、ルーティングに対しスロットリングが自動実行されます。

スロットリングの制御はipごと/会員ごとを指定可能です。

yamlファイルは、

- Customizeの場合、app/Customize/Resource/config/services.yaml
- プラグインの場合、app/Plugin/[Plugin Code]/Resource/config/services.yaml

に記載してください。
  
```yaml
eccube:
    rate_limiter:
        forgot:
            route: forgot # スロットリングを実行するルーティングを指定します。
            method: ['POST'] # スロットリングを実行するmethodを指定します。デフォルトはPOSTです。
            type: ip # スロットリングの制御方法を設定します。ip, customerを指定します。複数指定も可能です。
            limit: 5
            interval: '30 minutes'
        entry:
            route: entry
            method: ['POST']
            params:
                mode: complete # 会員登録画面のように、確認画面を挟む遷移をする場合、遷移用のパラメータを指定できます。
            type: [ 'ip', 'customer' ]
            limit: 5
            interval: '30 minutes'
        shopping_confirm_ip:
            route: ~ # routeがnullの場合は自動実行されません。
            limit: 25
            interval: '30 minutes'
```

### 独自のスロットリングを組み込みたい場合

routeにnullを指定することで、自動実行は行わず`RateLimiter`の生成のみ行うことができます。

```yaml
eccube:
    rate_limiter:
        hoge:
            route: ~ 
            limit: 25
            interval: '30 minutes'
```

コントローラ等、必要な箇所でRateLimiterFactoryをインジェクションし、実装してください。

```php
class HogeController {

  public function index(RateLimiterFactory $hogeLimiter, Request $request) {
    $limiter = $hogeLimiter->create($request->getClientIp());
    if (!$limiter->consume()->isAccepted()) {
        throw new TooManyRequestsHttpException()
    }
  }

```

### 既存の設定の上書きを行う

本体、もしくはプラグインの設定は、Customizeで上書きすることができます。

例えば、本体で以下のような設定がある場合、

```yaml
eccube:
 rate_limiter:
  forgot:
   route: forgot # スロットリングを実行するルーティングを指定します。
   method: ['POST'] # スロットリングを実行するmethodを指定します。デフォルトはPOSTです。
   type: ip # スロットリングの制御方法を設定します。ip, customerを指定します。複数指定も可能です。
   limit: 5
   interval: '30 minutes'
```

Customizeで以下のように上書きを行うことができます。

```yaml
eccube:
 rate_limiter:
  forgot:
   limit: 10
   interval: '60 minutes'
```

### スロットリングをクリアする

スロットリングの結果は、以下のコマンドでクリアできます。

```
bin/console cache:pool:clear rate_limiter.cache --env=<APP_ENV> 
```

### ログインスロットリング

ログインのスロットリング機能については、以下を参照してください。

- [EC-CUBE/ec-cube#4249](https://github.com/EC-CUBE/ec-cube/issues/4249)
- [EC-CUBE/ec-cube#5473](https://github.com/EC-CUBE/ec-cube/pull/5473)

## 参考情報

スロットリングの機能は、symfony/rate-limiterを利用しています。  
その他のカスタマイズ方法についてはSymfonyのドキュメントを参照してください。

[Rate Limiter](https://symfony.com/doc/5.4/rate_limiter.html){:target="_blank"}