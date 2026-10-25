# tab-maker

[alphaTab](https://alphatab.net) を使って、TAB 譜を表示・再生する試作ページです。

## alphaTab について

- 譜面の描画と再生には alphaTab（[coderline/alphaTab](https://github.com/coderline/alphaTab)）を使っています。このリポジトリは alphaTab の fork ではありません
- alphaTab 1.8.4 と音源の `sonivox.sf2` は、jsDelivr の CDN から読み込んでいます。リポジトリには同梱していません
- alphaTab のライセンスは [MPL-2.0](https://github.com/coderline/alphaTab/blob/develop/LICENSE) です

## 動かし方

`file://` で開くと再生が動かないことがあるため、ローカルサーバー経由で開きます。

```bash
python3 -m http.server 8000
```

起動したら http://localhost:8000 を開きます。
