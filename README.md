# vue-component-collection

このリポジトリは、Vue.jsを使用して様々なコンポーネントを実装し、追加していくためのプロジェクトです。各コンポーネントは再利用可能で、スタンドアロンで動作するように設計されています。

## 目次

- [コンポーネント一覧](#コンポーネント一覧)
- [使用方法](#使用方法)
- [貢献](#貢献)
- [ライセンス](#ライセンス)

## コンポーネント一覧

### DailyScheduler

1日予定表アプリを単一のVueコンポーネントとしてまとめました。このコンポーネントは、すべての機能を含む完全なスタンドアロンコンポーネントです。

![DailyScheduler](DailyScheduler/image-1.png)

詳細は[こちら](DailyScheduler/DailyScheduler.md)をご覧ください。

## 使用方法

各コンポーネントは、以下の手順でプロジェクトに組み込むことができます：

1. コンポーネントファイルをプロジェクトの`components`ディレクトリに保存します。
2. メインのVueファイルでインポートして使用します。

例：

```vue
<template>
  <div>
    <DailyScheduler />
  </div>
</template>

<script setup>
import DailyScheduler from './components/DailyScheduler.vue'
</script>