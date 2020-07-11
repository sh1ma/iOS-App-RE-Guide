# Appleのサンドボックス

Darwin(iOSやMacOSのOS基盤)のXNU(カーネル)にはサンドボックス機能があります．
サンドボックスというのはアプリケーションが実行できる操作を制限することにより，システムを保護する仕組みです．
ここではXNUのサンドボックスの設計と実装について説明します．

## 強制アクセス制御フレームワーク(MACF)

サンドボックス機能の基盤とも言えるのが__強制アクセス制御フレームワーク(MACF)__と言われるものです．英語では__Mandatory Access Control Framework__と表記します．

MACFは__強制アクセス制御(MAC)__，つまりセキュリティポリシー管理者によってユーザーに強制的に適用されるアクセス制御を容易に実装することができるフレームワークです．
実装，といっても私達が実装することはできません．何故ならiOSにおいて，セキュリティポリシー管理者はAppleだからです．

サンドボックスはMACFの__ポリシーモジュール__として実装されています．

## ポリシーモジュール

ポリシーモジュールとは，__ポリシー__の実装で，実体はkext(カーネル拡張)です．
そして，ここでいうポリシーというのは，特定の操作の制御に関する「ルールの集まり」のことです．

そして，サンドボックスは_sandbox.kext_という名前のポリシーモジュールです．
![](https://tva1.sinaimg.cn/large/007S8ZIlgy1ggivspc3dij326y05qabd.jpg)

_sandbox.kext_をはじめとしたポリシーモジュールは，(多くの場合，)カーネル上にロードされたときに，自身で関`mac_policy_register`を呼び出し，自身のポリシーをMACFに登録します．

[mac_policy_register - darwin-xnu/mac_base.c at master · apple/darwin-xnu](https://github.com/apple/darwin-xnu/blob/master/security/mac_base.c#L641) 

`mac_policy_register`はカーネルからエクスポートされた関数です．

### 登録の流れ

まず，ポリシーモジュールがロードされるとエントリーポイント`kmod_start`が発火し，内部で`mac_policy_register`が呼び出されます．この関数は[`mac_policy_conf`](https://github.com/apple/darwin-xnu/blob/master/security/mac_policy.h#L6708)という構造体へのポインタを引数に取ります．ここで重要なのは，`mac_policy_conf`内の[`mac_policy_ops`](https://github.com/apple/darwin-xnu/blob/master/security/mac_policy.h#L6292)という構造体のデータです．この構造体は`mpo_`から始まる型のメンバーを持っています．これはチェック関数のプレースホルダで，ポリシーモジュールがこれをフックすることでチェック機構が実装されます．そのうち[`mpo_policy_init`](https://github.com/apple/darwin-xnu/blob/master/security/mac_base.c#L778)，[`mpo_policy_initbsd`](https://github.com/apple/darwin-xnu/blob/master/security/mac_base.c#L782)の実装は`mac_policy_register`時に呼び出され，ポリシーを初期化します．

