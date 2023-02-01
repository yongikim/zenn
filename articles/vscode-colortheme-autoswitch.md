---
title: "Visual Studio Code のカラーテーマを OS の設定に合わせて切り替える方法"
emoji: "😀"
type: "tech"
topics: [VSCode,カラーテーマ]
published: false
---
OS の設定で、時間帯によってライトテーマとダークテーマを自動で切り替える設定を利用されている方に向けて、それぞれの時間帯で Visual Studio Code で利用するカラーテーマを指定する方法を紹介します。

結論から申し上げますと、`settings.json`を編集することで可能です。

例えばライトテーマのとき`Iceberg Light`、ダークテーマのとき`Iceberg`を利用したい場合は、

```json
{
  "window.autoDetectColorScheme": true,
  "workbench.preferredDarkColorTheme": "Iceberg",
  "workbench.preferredLightColorTheme": "Iceberg Light",
}
```

とすれば良いです。

