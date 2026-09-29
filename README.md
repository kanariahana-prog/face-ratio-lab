# face-ratio-lab
# Face Ratio Lab MVP v2

この試作品はMediaPipeのライブラリと顔ランドマークモデルをインターネットから読み込みます。

そのため、iPhoneの「ファイル」アプリからHTMLを直接開く `file://` 方式では、写真は表示されてもAI分析が動かない場合があります。

## 動かす方法
HTTPSのWebページとして公開して開いてください。

PCなら:
python3 -m http.server 8000

その後:
http://localhost:8000/

v2ではファイル選択の二重起動を修正し、写真プレビューと分析エラーを画面に明示するようにしました。
