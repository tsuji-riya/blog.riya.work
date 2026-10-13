# blog.riya.work

[Quartz 5](https://quartz.jzhao.xyz) で作ったブログ。`main` に push すると GitHub Pages にデプロイされる。

## 記事を書く

`content/` を Obsidian の Vault として開き、Markdown で書く。`content/index.md` がトップページ。

- 公開したくないノートは frontmatter に `draft: true`
- `private/` と `templates/` フォルダはビルド対象外

## ローカルで確認

```bash
npm ci
npx quartz build --serve   # http://localhost:8080
```

## 設定

`quartz.config.yaml` を編集する（タイトル・フォント・色・プラグインなど）。

コメント欄(giscus)を使う場合は https://giscus.app で `repoId` / `categoryId` を取得して、`@quartz-community/comments` の `TODO:` を埋め、`enabled: true` にする。

## 初回デプロイ設定

1. GitHub の Settings → Pages → Source を **GitHub Actions** にする
2. Settings → Pages → Custom domain に `blog.riya.work` を入力して保存（Enforce HTTPS もオン）
3. DNS に `blog` の CNAME レコードを `tsuji-riya.github.io` 向けで追加
