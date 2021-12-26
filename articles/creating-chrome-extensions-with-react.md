---
title: "React で作る Chrome 拡張機能"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["React", "TypeScript", "Chrome拡張機能"]
published: false
---

Chrome 拡張機能を使うことで、ページ内に新しい機能を追加したり、情報を取得したりと色々できます。

自分が使い慣れている React を使って Chrome 拡張機能を作ってみたので、その作り方を簡単にまとめておこうと思います。
また、 Google 拡張機能の詳しい書き方については言及しないこととします。

今回は例として、サンプリアプリ (`sample_app`) という名前のアプリを作っていきます。


# TL;DR
この記事で作成したプログラムは以下のリポジトリに置いてあります。

[ulcttku/sample_chrome_extension_with_react](https://github.com/ulcttku/sample_chrome_extension_with_react)

もしよければ PR などいただければ幸いです。


# 実装

## 準備

### React アプリの作成

まずは、 `create-react-app` を使って React の開発環境を整えます。
`--template` の後ろは `TypeScript` ではなく `typescript` でないとエラーになります。

```bash
> yarn create react-app sample_app --template typescript
```

念の為、中身を確認しておきます。

```bash
> cd sample_app
> ls
README.md  node_modules  package-lock.json  public  tsconfig.json
build      null          package.json       src
> ls src
App.css       App.tsx    index.tsx  react-app-env.d.ts  setupTests.ts
App.test.tsx  index.css  logo.svg   reportWebVitals.ts
```

ちゃんと TypeScript が使われていそうですね。

ついでに、きちんと動くかも確認しておきます。
以下のコマンドを実行して、`http://localhost:3000` が開かれば成功です。

```bash
> yarn start
```

確認できたら、`Ctrl + C` でジョブを終了しておきます。


### ライブラリのインストール

実際に Chrome 拡張機能を TypeScript で作成するときに必要となるので、型情報を取得しておきます。

```bash
> yarn add -D @types/chrome
```

### `manifest.json` の作成

`public` フォルダにある `manifest.json` を上書きします。

詳しい書き方や説明は、以下のリンクなどをご参照ください。

* [Manifest file format - Chrome Developers](https://developer.chrome.com/docs/extensions/mv3/manifest/)
* [Chrome 拡張機能のマニフェストファイルの書き方 - Qiita](https://qiita.com/mdstoy/items/9866544e37987337dc79) (※ ただし version 2)



一旦、最低限必要な `manifest.json` を書いておきます。

```json:manifest.json
{
  "manifest_version": 3,
  "name": "Sample App",
  "version": "0.0.1",

  "action": {
    "default_title": "サンプルアプリ"
  },
  "description": "React で Chrome 拡張機能を作るためのサンプル"
}
```

## popup の作成

拡張機能のアイコンをクリックしたときに表示されるポップアップを作っていきます。

### ビルドの準備

まずは、`manifest.json` に `default_popup` を追加します。

```git:manifest.json
 {
   "manifest_version": 3,
   "name": "Sample App",
   "version": "0.0.1",

   "action": {
-    "default_title": "サンプルアプリ"
+    "default_title": "サンプルアプリ",
+    "default_popup": "popup.html"
   },
   "description": "React で Chrome 拡張機能を作るためのサンプル"
 }
```

React のビルドに使用している `react-script` はビルド結果を `index.html` から変更できないので、`popup.html` を作成するには少し工夫が必要です。
今後、オプションページを使うかもしれない事も考えて、以下のように書き換えていきます。

```git:package.json
   "scripts": {
     "start": "react-scripts start",
-    "build": "react-scripts build",
+    "build": "react-scripts build && yarn rename:popup",
+    "rename:popup": "sed 's/root/popup/' build/index.html > build/popup.html",
     "test": "react-scripts test",
     "eject": "react-scripts eject"
   },
```

```git:src/index.tsx
 ReactDOM.render(
   <React.StrictMode>
     <App />
   </React.StrictMode>,
-  document.getElementById('root')
+  document.getElementById('popup')
);
```

### 試運転

それでは Chrome 拡張機能として動かしてみましょう!

まずは、ビルドします。

```bash
> yarn build
```

念の為、どうなったか確認しておきます。

```bash
>ls build
asset-manifest.json  index.html   logo512.png    popup.html  static
favicon.ico          logo192.png  manifest.json  robots.txt
```

大丈夫そうですね。


では、Chrome で `chrome://extensions/` を開いて、右上にあるデベロッパーモードをオンにします。

![](/images/creating-chrome-extensions-with-react/extensions.png)

デベロッパーモードをオンにすると、上の方にボタンが表示され、各拡張機能にIDなど追加の情報が表示されます。

![](/images/creating-chrome-extensions-with-react/developer_mode.png)


次に ｢パッケージ化されていない拡張機能を読み込む｣をクリックして、ビルドしたフォルダを選択します。
以下の画像のように表示されれば成功です。
ID は環境に変わると思うので気にしなくて大丈夫です。

![](/images/creating-chrome-extensions-with-react/load_sample_app.png)

最後に、読み込ませた拡張機能をツールバーにピン留めしておきます。
ツールバーにあるパズルアイコンをクリックして、｢Sample App｣の右横にある押しピンアイコンをクリックします。

![](/images/creating-chrome-extensions-with-react/fix_chrome_extension.png)

ツールバーに｢S｣と書かれたアイコンが表示されるので、そのアイコンをクリックして React のサンプルページが表示されれば成功です!!

![](/images/creating-chrome-extensions-with-react/show_react_sample_page.png)

### Popup コンポーネントの実装

では、実際にポップアップに表示させるコンテンツを実装してみましょう。

今回は例なので、簡単に、`Hello Chrome Extensions` と表示するだけのものを作ります。

まずは、`src/index.tsx` を変更します。

```git:src/index.tsx
 import React from 'react';
 import ReactDOM from 'react-dom';
 import './index.css';
 import App from './App';
+import Popup from './Popup';
 import reportWebVitals from './reportWebVitals';

 ReactDOM.render(
   <React.StrictMode>
-    <App />
+    <Popup />
   </React.StrictMode>,
-  document.getElementById('root')
+  document.getElementById('popup')
 );
```

最後に、`src/Popup/index.tsx` を作成します。

```tsx:src/Popup/index.tsx
import React from 'react';

const Popup: React.VFC = () => {
    return (
        <div style={{width: '200px'}}>
            Hello Chrome Extensions
        </div>
    );
}

export default Popup;
```

そうしたら、ビルドしてブラウザに読み込ませます。

Chrome で `chrome://extensions/` を開いて、Sample App を再読み込みをします。

Sample App のカードの右下にある更新マークをクリックします。
画面左下に｢再読み込されました｣と表示されれば大丈夫です。

![](/images/creating-chrome-extensions-with-react/reload_sample_app.png)

そうしたら Sample App のアイコンをクリックして、`Hello Chrome Extensions` と表示されば成功です!!


## options の作成

拡張機能の設定などをするためのページとして、オプションページを作っていきます。

### ビルドの準備

まずは、`manifest.json` に `options_page` を追加します。

```git:manifest.json
 {
   "manifest_version": 3,
   "name": "Sample App",
   "version": "0.0.1",

   "action": {
     "default_title": "サンプルアプリ",
     "default_popup": "popup.html"
   },
-  "description": "React で Chrome 拡張機能を作るためのサンプル"
+  "description": "React で Chrome 拡張機能を作るためのサンプル",
+  "options_page": "options.html"
}
```

次に、`package.json` を変更します。

```git:package.json
   "scripts": {
     "start": "react-scripts start",
-    "build": "react-scripts build && yarn rename:popup",
+    "build": "react-scripts build && yarn rename:popup && yarn rename:options",
     "rename:popup": "sed 's/root/popup/' build/index.html > build/popup.html",
+    "rename:options": "sed -e 's/root/options/' build/index.html >  build/options.html",
     "test": "react-scripts test",
     "eject": "react-scripts eject"
   },
```

最後に `src/index.tsx` で `options` にもレンダーするように変更しておきます。

```git:src/index.tsx
 import React from 'react';
 import ReactDOM from 'react-dom';
 import './index.css';
 import App from './App';
 import Popup from './Popup';
 import reportWebVitals from './reportWebVitals';

 ReactDOM.render(
   <React.StrictMode>
     <Popup />
   </React.StrictMode>,
-  document.getElementById('popup')
+  document.getElementById('popup') || document.createElement('div')
 );
+
+ReactDOM.render(
+  <React.StrictMode>
+    <App />
+  </React.StrictMode>,
+  document.getElementById('options') || document.createElement('div')
+);
```

ここで `document.createElement('div')` がないと、オプションページを開いた際に `Target container is not a DOM element.` というエラーが表示されてしまうので追加しています。


### 試運転

それではまた動かしてみましょう!

まずは、ビルドします。

```bash
> yarn build
```

念の為、どうなったか確認しておきます。

```bash
>ls build
asset-manifest.json  index.html   logo512.png    options.html  robots.txt
favicon.ico          logo192.png  manifest.json  popup.html    static
```

ちゃんと `options.html` もありますね。


では、拡張機能を再読み込みして Sample App の詳細から、拡張機能の詳細ページを開きます。
そして、｢拡張機能のオプション｣をクリックして React のサンプルページが表示されれば成功です!!

### Options コンポーネントの実装

では、実際にオプションページに表示させるコンテンツを実装してみましょう。

今回は例なので、簡単に、`Here is Options Page.` と表示するだけのものを作ります。

まずは、`src/index.tsx` を変更します。

```git:src/index.tsx
 import React from 'react';
 import ReactDOM from 'react-dom';
 import './index.css';
 import App from './App';
+import Options from './Options';
 import Popup from './Popup';
 import reportWebVitals from './reportWebVitals';

 ReactDOM.render(
   <React.StrictMode>
-    <App />
+    <Popup />
   </React.StrictMode>,
-  document.getElementById('root')
+  document.getElementById('popup')
 );
```

最後に、`src/Options/index.tsx` を作成します。

```tsx:src/Popup/index.tsx
import React from 'react';

const Options: React.VFC = () => {
    return (
        <div>
            Here is Options Page.
        </div>
    );
}

export default Options;
```

そしたら、再読み込みをした上で、拡張機能のオプションページを開いて、`Here is Options Page.` と表示されば成功です!!

## Oprion と popup の連携

ここはあまり必要ないのですが、オプションページで設定した内容をポップアップに反映できたほうが実際に使うときと近い実装にできると思うので、やっておきたいと思います。

### 準備
まずは、オプションページで設定した内容を保存するために、`chrome.storage` を使えるように permissions を設定します。

```git:public/manifest.json
     "default_popup": "popup.html"
   },
   "description": "React で Chrome 拡張機能を作るためのサンプル",
-  "options_page": "options.html"
+  "options_page": "options.html",
+  "permissions": [
+    "storage"
+  ]
 }
```

あとは、`src/Options/index.tsx` と `src/Popup/index.tsx` を以下のように変更します。

```tsx:src/Options/index.tsx
import React, { useEffect } from 'react';

const Options: React.VFC = () => {
    useEffect(() => {
        chrome.storage.local.get(['greeting'], (result) => {
            const element = document.querySelector('input');
            element && element.setAttribute('value', result.greeting);
        });
    }, []);

    const save = () => {
        chrome.storage.local.set({
            greeting: document.querySelector('input')?.value || 'Hello',
        });
        alert('保存しました');
    }

    return (
        <div>
            <label htmlFor='greeting'>挨拶: </label>
            <input id='greeting' />
            <button onClick={save}>保存</button>
        </div>
    );
}

export default Options;
```

```tsx:src/Popup/index.tsx
import React, { useEffect, useState } from 'react';

const Popup: React.VFC = () => {
    const [greeting, setGreeting] = useState('');
    useEffect(() => {
        chrome.storage.local.get(['greeting'], (result) => {
            setGreeting(result.greeting);
        });
    }, []);

    return (
        <div style={{ width: '200px' }}>
            {greeting} Chrome Extensions
        </div>
    );
}

export default Popup;
```

あとは、ビルドして、再読み込みすれば動作の確認ができるかと思います。

オプションページに適当な挨拶を入力して、保存すればポップアップにその挨拶が表示されるようになるかと思います。

本当は、デフォルト値の設定だったり色々やるべきことはあるかと思いますが、サンプルなのでこれぐらいの実装にしておきます。


## content_scripts の作成

拡張機能からブラウザで開いているページにアクセスするための、`content_scripts` というものを作っていきます。


### ビルドの準備

`content_scripts` はただの JavaScript ファイルなので、 React と同じビルドではなく Webpack を使って別途ビルドします。



まずは、必要なライブラリをインストールします。

```bash
> yarn add -D ts-loader ts-node webpack-cli
```

次に、`tsconfig.json` を変更します。 `tsconfig.json` は React をビルドする際に使われるので、`content_scripts/` 内のファイルが影響しないように `exclude` を設定しておきます。
ここでは、`exclude` を有効にしたときの初期値と `src/content_scripts` を指定しています。

また、後述する `webpack.content_scripts.config.ts` をうまく読み込ませるために `ts-node` の設定もします。
詳しい `ts-node` の設定については [Configuration Languages | webpack](https://webpack.js.org/configuration/configuration-languages/) をご参照ください。

```git:tsconfig.json
   },
   "include": [
     "src"
-  ]
+  ],
+  "exclude": [
+    "node_modules",
+    "bower_components",
+    "jspm_packages",
+    "src/content_scripts"
+  ],
+  "ts-node": {
+    "compilerOptions": {
+      "module": "CommonJS"
+    }
+  }
 }

```

次に、`content_scripts` 用の `tsconfig.content_scripts.json` を作成します。
というのも、デフォルトの `tsconfig.json` だと `noEmit` が `true` となっているので、そのままではビルド結果を出力することができません。

```json:tsconfig.content_scripts.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": false,
  },
  "include": [
    "src/content_scripts"
  ],
  "exclude": [
    "node_modules",
    "bower_components",
    "jspm_packages"
  ],
}
```

次に、`content_scripts` 用の `webpack.content_scripts.config.ts` を作ります。
詳しい Webpack の設定内容については [Configuration | webpack](https://webpack.js.org/configuration/) をご参照ください。

```ts:webpack.content_scripts.config.ts
const path = require('path');
module.exports = {
    mode: 'production',
    entry: './src/content_scripts/index.ts',
    output: {
        path: path.resolve(__dirname, 'build'),
        filename: 'content_script.js',
    },
    module: {
        rule: [
            {
                test: /\.ts$/,
                use: {
                    loader: 'ts-loader',
                    options: {
                        transpileOnly: true,
                        configFile: 'tsconfig.content_scripts.json',
                    }
                },
            },
        ],
    },
    resolve: {
        extensions: [
            '.ts', '.js',
        ],
    },
}
```

次に、`package.json` を変更します。

```git:package.json
   "scripts": {
     "start": "react-scripts start",
-    "build": "react-scripts build && yarn rename:popup && yarn rename:options",
+    "build": "yarn build:react && yarn build:content_scripts",
+    "build:react": "react-scripts build && yarn rename:popup && yarn rename:options",
+    "build:content_scripts": "webpack --config webpack.content_scripts.config.ts",
     "rename:popup": "sed 's/root/popup/' build/index.html > build/popup.html",
     "rename:options": "sed -e 's/root/options/' build/index.html >  build/options.html",
     "test": "react-scripts test",
     "eject": "react-scripts eject"
   },
```

最後に、空の TypeScript ファイル: `src/content_scripts/index.ts` を作成します。

### 試運転

今回は `src/content_scripts/index.ts` の中身をまだ書いていないので、ビルドできるかだけ確認しておきます。

コマンドだけ載せるのも味気ないので、今回だけビルド結果まで載せておきます。

```bash
> yarn build
yarn run v1.22.10
$ yarn build:react && yarn build:content_scripts
$ react-scripts build && yarn rename:popup && yarn rename:options
Creating an optimized production build...
Compiled with warnings.

src\index.tsx
  Line 4:8:  'App' is defined but never used  @typescript-eslint/no-unused-vars

Search for the keywords to learn more about each warning.
To ignore, add // eslint-disable-next-line to the line before.

File sizes after gzip:

  44.2 kB  build\static\js\main.629fb95d.js
  1.78 kB  build\static\js\787.90d91508.chunk.js
  264 B    build\static\css\main.e6c13ad2.css

The project was built assuming it is hosted at /.
You can control this with the homepage field in your package.json.

The build folder is ready to be deployed.
You may serve it with a static server:

  yarn global add serve
  serve -s build

Find out more about deployment here:

  https://cra.link/deployment

$ sed 's/root/popup/' build/index.html > build/popup.html
$ sed -e 's/root/options/' build/index.html >  build/options.html
$ webpack --config webpack.content_scripts.config.ts
asset content_scripts.js 43 bytes [emitted] [minimized] (name: main)
./src/content_scripts/index.ts 44 bytes [built] [code generated]
webpack 5.65.0 compiled successfully in 275 ms
Done in 11.76s.
```

問題なくビルドできていそうですね。

### content_scripts の実装

では、実際に content_scripts を実装してみましょう。

今回は簡単のために、ボタンをクリックすることで開いているページのタイトルを取得する処理を書いてみます。

まずは、`public/manifest.json` に `content_script.js` をすべてのサイトで使用できるように設定します。
次に、あとで必要になるので、`permissions` に `tabs` を追加しておきます。

```git:public/manifest.json
     "default_title": "サンプルアプリ",
     "default_popup": "popup.html"
   },
+  "content_scripts": [
+    {
+      "matches": ["http://*/*", "https://*/*"],
+      "js": ["content_script.js"]
+    }
+  ],
   "description": "React で Chrome 拡張機能を作るためのサンプル",
   "options_page": "options.html",
   "permissions": [
-    "storage"
+    "storage",
+    "tabs"
   ]
 }
```

次に、 `src/Popup/index.tsx` にボタンを追加して、クリックした際にタイトルを表示するを書きます。

```tsx:src/Popup/index.tsx
import React, { useEffect, useState } from 'react';

const Popup: React.VFC = () => {
    const [greeting, setGreeting] = useState('');
    const [title, setTitle] = useState('');

    useEffect(() => {
        chrome.storage.local.get(['greeting', 'title'], (result) => {
            setGreeting(result.greeting);
            setTitle(result.title);
        });
    }, []);

    const getTitle = () => {
        chrome.tabs.query({ active: true, currentWindow: true }, (tabs) => {
            const tabId  =tabs[0].id;
            if(!tabId) {
                setTitle('NotTitle');
                return;
            }

            chrome.tabs.sendMessage(tabId, { type: 'getTitle' }, (response) => {
                chrome.storage.local.set({ title: response.title });
                setTitle(response.title);
            });
        });
    }

    return (
        <div style={{ width: '200px' }}>
            <div>
                {`${greeting} ｢${title}｣`}
            </div>

            <button onClick={getTitle}>タイトルを取得</button>
        </div>
    );
}

export default Popup;
```

次に、 `src/content_scripts/index.ts` にタイトルを取得するを書きます。
```ts:src/content_scripts/index.ts
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
    if(message.type !== 'getTitle') return;

    sendResponse({
        title: document.querySelector('title').innerText,
    })
})
```

これで実装は完了したので、再読み込みをしてください。
あとは、適当なページを開いてからアイコンをクリックして、｢タイトルを取得｣をクリックして `｢｣` 内にページのタイトルが表示されれば成功です!!

もしかしたら表示されない場合があるかもしれませんが、そのときはページを更新してみてください。


## あとがき

ほかにも不要なファイルを削除したり、アイコンの設定をしたり、`backgroud.js` を作成したり色々やることがありますが、気力が尽きたのでここまでとします…。
