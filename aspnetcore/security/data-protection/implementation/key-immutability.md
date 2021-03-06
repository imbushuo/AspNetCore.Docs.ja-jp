---
title: ASP.NET Core のキーの不変性とキー設定
author: rick-anderson
description: ASP.NET Core データ保護のキー不変 Api の実装の詳細について説明します。
ms.author: riande
ms.date: 10/14/2016
no-loc:
- Blazor
- Identity
- Let's Encrypt
- Razor
- SignalR
uid: security/data-protection/implementation/key-immutability
ms.openlocfilehash: 5bbd1fb28fc0330ccfe4e829ab0b2f86226c7166
ms.sourcegitcommit: 70e5f982c218db82aa54aa8b8d96b377cfc7283f
ms.translationtype: MT
ms.contentlocale: ja-JP
ms.lasthandoff: 05/04/2020
ms.locfileid: "82776917"
---
# <a name="key-immutability-and-key-settings-in-aspnet-core"></a>ASP.NET Core のキーの不変性とキー設定

オブジェクトがバッキングストアに永続化されると、その表現は永久に固定されます。 バッキングストアに新しいデータを追加することはできますが、既存のデータを変換することはできません。 この動作の主な目的は、データの破損を防ぐことです。

この動作の1つの結果として、キーがバッキングストアに書き込まれると、変更できなくなります。 作成、アクティブ化、および有効期限の日付は変更できませんが、を使用`IKeyManager`して取り消すことができます。 さらに、基になるアルゴリズム情報、マスターキーマテリアル、および保存時の暗号化プロパティも変更できません。

キーの永続化に影響する設定が開発者によって変更された場合、これらの変更は、次に`IKeyManager.CreateNewKey`キーが生成されるまで有効になりません。これは、またはデータ保護システム独自の[自動キー生成](xref:security/data-protection/implementation/key-management#data-protection-implementation-key-management)動作を使用した明示的な呼び出しによって行われます。 キーの永続化に影響する設定は次のとおりです。

* [既定のキーの有効期間](xref:security/data-protection/implementation/key-management#data-protection-implementation-key-management)

* [保存時のキー暗号化メカニズム](xref:security/data-protection/implementation/key-encryption-at-rest)

* [キー内に含まれるアルゴリズム情報](xref:security/data-protection/configuration/overview#changing-algorithms-with-usecryptographicalgorithms)

これらの設定を次の自動キーのローリングタイムより前に開始する必要がある場合は、を`IKeyManager.CreateNewKey`明示的に呼び出して新しいキーを強制的に作成することを検討してください。 明示的なアクティベーション日 ({now + 2 日} は、変更が反映されるまでの時間を確保するための適切な目安です) と通話の有効期限を指定してください。

>[!TIP]
> リポジトリに接しているすべてのアプリケーションでは、 `IDataProtectionBuilder`拡張メソッドで同じ設定を指定する必要があります。 それ以外の場合、永続化されたキーのプロパティは、キー生成ルーチンを呼び出した特定のアプリケーションに依存します。
