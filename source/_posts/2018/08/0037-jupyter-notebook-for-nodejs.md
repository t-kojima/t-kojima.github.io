---
title: 'Node.js でも Jupyter Notebook が使いたい'
date: 2018-08-09 15:27:32
tags:
  - jupyter
  - nodejs
---

Node.js でも Jupyter Notebook が使いたい！

![jupyter logo](/images/37-jupyter.png)

<!-- more -->

# 環境

- Windows 10
- Python 3.6.3
- Node 8.11.3

`※Python と Node は既にインストール済み`

# jupyter-nodejs

手始めに調べたところヒットしたのが以下のリポジトリだ。

<a href="https://github.com/notablemind/jupyter-nodejs" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

Installation に従って進めるが、Windows 環境だと `yarn install` のタイミングで以下のエラーになる。

```
info There appears to be trouble with your network connection. Retrying...
[4/4] Building fresh packages...
[1/1] ⠄ zmq: not ok
[-/1] ⠄ waiting...
[-/1] ⠄ waiting...
[-/1] ⡀ waiting...
error An unexpected error occurred: "C:\\Users\\********\\Documents\\Git\\JavaScript\\jupyter-nodejs\
ode_modules\\zmq: Command failed.
Exit code: 1
Command: C:\\Windows\\system32\\cmd.exe
Arguments: /d /s /c node-gyp rebuild
Directory: C:\\Users\\********\\Documents\\Git\\JavaScript\\jupyter-nodejs\
ode_modules\\zmq
Output:
C:\\Users\\********\\Documents\\Git\\JavaScript\\jupyter-nodejs\
ode_modules\\zmq>if not defined npm_config_node_gyp (node \"C:\\Program Files\
odejs\
ode_modules\
pm\\bin\
ode-gyp-bin\\\\..\\..\
ode_modules\
ode-gyp\\bin\
ode-gyp.js\" rebuild )  else (node \"\" rebuild )
gyp info it worked if it ends with ok
gyp info using node-gyp@3.6.2
gyp info using node@8.11.3 | win32 | x64

...以下省略
```

zmq がごにょごにょ…という感じで、ふんいきから感じ取ると恐らく `zeromq` のモジュールが無い感じだ。

探すと Windows 向けにも[ありそう](http://zeromq.org/distro:microsoft-windows)なんだけど、これを入れて解決するかは不明なので、WSL でやることにする。

ちなみに zeromq が無いとエラーになるので事前にインストールしておく。

```bash
sudo apt-get install libzmq-dev
```

そしてインストール手順に従って進めていく。

```bash
git clone https://github.com/notablemind/jupyter-nodejs.git
cd jupyter-nodejs
mkdir -p ~/.ipython/kernels/nodejs/
npm install && node install.js
npm run build
npm run build-ext
```

インストール自体は問題なくいけた。

しかし、`ipython console --kernel nodejs` でやはりエラーになる。

```
$ ipython console --kernel nodejs
Traceback (most recent call last):
  File "/usr/bin/ipython", line 6, in <module>
    start_ipython()
  File "/usr/lib/python2.7/dist-packages/IPython/__init__.py", line 118, in start_ipython
    return launch_new_instance(argv=argv, **kwargs)
  File "/usr/lib/python2.7/dist-packages/IPython/config/application.py", line 545, in launch_instance
    app.initialize(argv)
  File "<string>", line 2, in initialize
  File "/usr/lib/python2.7/dist-packages/IPython/config/application.py", line 89, in catch_config_error
    return method(app, *args, **kwargs)
  File "/usr/lib/python2.7/dist-packages/IPython/terminal/ipapp.py", line 312, in initialize
    super(TerminalIPythonApp, self).initialize(argv)
  File "<string>", line 2, in initialize
  File "/usr/lib/python2.7/dist-packages/IPython/config/application.py", line 89, in catch_config_error
    return method(app, *args, **kwargs)
  File "/usr/lib/python2.7/dist-packages/IPython/core/application.py", line 373, in initialize
    self.parse_command_line(argv)
  File "/usr/lib/python2.7/dist-packages/IPython/terminal/ipapp.py", line 307, in parse_command_line
    return super(TerminalIPythonApp, self).parse_command_line(argv)
  File "<string>", line 2, in parse_command_line
  File "/usr/lib/python2.7/dist-packages/IPython/config/application.py", line 89, in catch_config_error
    return method(app, *args, **kwargs)
  File "/usr/lib/python2.7/dist-packages/IPython/config/application.py", line 474, in parse_command_line
    return self.initialize_subcommand(subc, subargv)
  File "<string>", line 2, in initialize_subcommand
  File "/usr/lib/python2.7/dist-packages/IPython/config/application.py", line 89, in catch_config_error
    return method(app, *args, **kwargs)
  File "/usr/lib/python2.7/dist-packages/IPython/config/application.py", line 405, in initialize_subcommand
    subapp = import_item(subapp)
  File "/usr/lib/python2.7/dist-packages/IPython/utils/importstring.py", line 42, in import_item
    module = __import__(package, fromlist=[obj])
  File "/usr/lib/python2.7/dist-packages/IPython/terminal/console/app.py", line 25, in <module>
    from IPython.consoleapp import (
  File "/usr/lib/python2.7/dist-packages/IPython/consoleapp.py", line 35, in <module>
    from IPython.kernel.blocking import BlockingKernelClient
  File "/usr/lib/python2.7/dist-packages/IPython/kernel/__init__.py", line 4, in <module>
    from . import zmq
  File "/usr/lib/python2.7/dist-packages/IPython/kernel/zmq/__init__.py", line 14, in <module>
    check_for_zmq('2.1.11', 'IPython.kernel.zmq')
  File "/usr/lib/python2.7/dist-packages/IPython/utils/zmqrelated.py", line 37, in check_for_zmq
    raise ImportError("%s requires python-zmq >= %s"%(required_by, minimum_version))
ImportError: IPython.kernel.zmq requires python-zmq >= 2.1.11
```

`pip install pyzmq` してもこのエラーは解消しない。

というかこれは仮に動いても Github の README を見る限り、なんか ipython で nodejs が動くようなふんいきだ。

僕がやりたいのはブラウザで動くタイプの Jupyter Notebook なんです！

# ijavascript

次の選択肢はこれ、こちらはちゃんとローカルサーバーが立って Jupyter Notebook が使えるふんいきだ。

<a href="https://github.com/n-riesco/ijavascript" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

これも Installation に従って進める

まず pipenv を使って、venv の仮想環境に jupyter を入れる。

```bash
pipenv install jupyter
pipenv shell
```

つぎに ijavascript を yarn でインストール。

```bash
yarn init
yarn add ijavascript
yarn ijsinstall
```

起動！

```bash
jupyter notebook
```

するとローカルサーバーが立ちあがり、Node.js が追加された Jupyter Notebook を使うことができる。

![Node.jsが選択できるようになっている](/images/37-01.png)

## スクリプトの実行で Kernel Error

しかし実際ノートブックを書いていると、JavaScript が動作せず、`Kernel Error`というエラーが出ていた。

```
[E 10:56:36.735 NotebookApp] Failed to run command:
    ['ijskernel.cmd', '--hide-undefined', 'C:\\Users\\********\\AppData\\Roaming\\jupyter\\runtime\\kernel-0330c836-ab32-4e7b-bb1f-758b26fa4375.json', '--protocol=5.0']
        PATH='C:\\Users\\********\\Documents\\Git\\JavaScript\\jupyter\\.venv\\Scripts;C:\\ProgramData\\Oracle\\Java\\javapath;C:\\Program Files\\Docker\\Docker\\Resources\\bin;C:\\Program Files\\Microsoft MPI\\Bin\\;C:\\Windows\\system32;C:\\Windows;C:\\Windows\\System32\\Wbem;C:\\Windows\\System32\\WindowsPowerShell\\v1.0\\;C:\\Program Files (x86)\\Microsoft SQL Server\\140\\Tools\\Binn\\;C:\\Program Files\\Microsoft SQL Server\\140\\Tools\\Binn\\;C:\\Program Files (x86)\\Microsoft SQL Server\\140\\DTS\\Binn\\;C:\\Program Files\\Microsoft SQL Server\\140\\DTS\\Binn\\;C:\\Program Files\\Microsoft SQL Server\\Client SDK\\ODBC\\130\\Tools\\Binn\\;C:\\Program Files (x86)\\Microsoft SQL Server\\Client SDK\\ODBC\\130\\Tools\\Binn\\;C:\\Program Files (x86)\\Microsoft SQL Server\\140\\Tools\\Binn\\ManagementStudio\\;C:\\Program Files\\Microsoft SQL Server\\130\\Tools\\Binn\\;C:\\Users\\********\\AppData\\Local\\Programs\\Python\\Python36-32\\Scripts\\;C:\\Users\\********\\AppData\\Local\\Programs\\Python\\Python36-32\\;C:\\Users\\********\\AppData\\Local\\Microsoft\\WindowsApps;C:\\Program Files\\Microsoft VS Code\\bin;C:\\Ruby\\Tools\\uru;C:\\Ruby\\Ruby23\\bin;C:\\Users\\********\\AppData\\Roaming\\npm;C:\\Program Files\\OpenSSH;C:\\Program Files\\Git\\cmd;C:\\ProgramData\\chocolatey\\bin;C:\\Program Files\\nodejs\\;C:¥MinGW¥bin;C:\\Users\\********\\AppData\\Roaming\\local\\bin;C:\\Users\\********\\AppData\\Local\\Programs\\Python\\Python36-32\\Scripts\\;C:\\Users\\********\\AppData\\Local\\Programs\\Python\\Python36-32\\;C:\\Users\\********\\AppData\\Local\\Microsoft\\WindowsApps;C:\\Ruby\\Tools\\uru;C:\\Ruby\\Ruby23\\bin;C:\\Program Files\\cli-kintone;C:\\Program Files\\Microsoft VS Code\\bin;C:\\Program Files (x86)\\phantomjs-2.1.1-windows\\bin;C:\\Program Files (x86)\\chromedriver_win32;C:\\Program Files\\wkhtmltopdf\\bin;C:\\Program Files\\wkhtmltopdf\\bin\\wkhtmltoimage.exe;C:\\tools\\cmder;C:\\Users\\********\\AppData\\Roaming\\npm;c:\\users\\********\\appdata\\local\\programs\\python\\python36-32\\lib\\site-packages\\pywin32_system32'
        with kwargs:
    {'stdin': -1, 'stdout': None, 'stderr': None, 'cwd': 'C:\\Users\\********\\Documents\\Git\\JavaScript\\jupyter'}

[E 10:56:36.737 NotebookApp] Uncaught exception POST /api/sessions (127.0.0.1)
    HTTPServerRequest(protocol='http', host='localhost:8888', method='POST', uri='/api/sessions', version='HTTP/1.1', remote_ip='127.0.0.1')
    Traceback (most recent call last):
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\web.py", line 1592, in _execute
        result = yield result
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\gen.py", line 1133, in run
        value = future.result()
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\gen.py", line 1141, in run
        yielded = self.gen.throw(*exc_info)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\notebook\services\sessions\handlers.py", line 73, in post
        type=mtype))
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\gen.py", line 1133, in run
        value = future.result()
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\gen.py", line 1141, in run
        yielded = self.gen.throw(*exc_info)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\notebook\services\sessions\sessionmanager.py", line 79, in create_session
        kernel_id = yield self.start_kernel_for_session(session_id, path, name, type, kernel_name)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\gen.py", line 1133, in run
        value = future.result()
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\gen.py", line 1141, in run
        yielded = self.gen.throw(*exc_info)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\notebook\services\sessions\sessionmanager.py", line 92, in start_kernel_for_session
        self.kernel_manager.start_kernel(path=kernel_path, kernel_name=kernel_name)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\gen.py", line 1133, in run
        value = future.result()
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\tornado\gen.py", line 326, in wrapper
        yielded = next(result)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\notebook\services\kernels\kernelmanager.py", line 160, in start_kernel
        super(MappingKernelManager, self).start_kernel(**kwargs)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\jupyter_client\multikernelmanager.py", line 110, in start_kernel
        km.start_kernel(**kwargs)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\jupyter_client\manager.py", line 259, in start_kernel
        **kw)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\jupyter_client\manager.py", line 204, in _launch_kernel
        return launch_kernel(kernel_cmd, **kw)
      File "c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\jupyter_client\launcher.py", line 128, in launch_kernel
        proc = Popen(cmd, **kwargs)
      File "c:\users\********\appdata\local\programs\python\python36-32\Lib\subprocess.py", line 709, in __init__
        restore_signals, start_new_session)
      File "c:\users\********\appdata\local\programs\python\python36-32\Lib\subprocess.py", line 997, in _execute_child
        startupinfo)
    FileNotFoundError: [WinError 2] 指定されたファイルが見つかりません。
[W 10:56:36.750 NotebookApp] Unhandled error
[E 10:56:36.751 NotebookApp] {
      "Host": "localhost:8888",
      "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:61.0) Gecko/20100101 Firefox/61.0",
      "Accept": "application/json, text/javascript, */*; q=0.01",
      "Accept-Language": "ja,en-US;q=0.7,en;q=0.3",
      "Accept-Encoding": "gzip, deflate",
      "Referer": "http://localhost:8888/notebooks/Tilde_and_IndexOf.ipynb",
      "Content-Type": "application/json",
      "X-Xsrftoken": "2|4dcc5b30|03fde516121f595f281c8c0fe4867c74|1533774838",
      "X-Requested-With": "XMLHttpRequest",
      "Content-Length": "103",
      "Cookie": "username-localhost-8888=\"2|1:0|10:1533779787|23:username-localhost-8888|44:NDJlN2ExNGFkMzliNGY4YjlkNDNiMWYxODg5NTE1YWM=|0623f8e34931a73fea2928b271be35e69dea1d11d513d74d0b3b0b4a8a76275b\"; _xsrf=2|4dcc5b30|03fde516121f595f281c8c0fe4867c74|1533774838",
      "Dnt": "1",
      "Connection": "keep-alive"
    }
[E 10:56:36.751 NotebookApp] 500 POST /api/sessions (127.0.0.1) 63.00ms referer=http://localhost:8888/notebooks/Tilde_and_IndexOf.ipynb
```

調べるとカーネルのパスに問題があると同様のエラーになるらしい。

以下が現在の状態のパスだ

```bash
$ jupyter kernelspec list
Available kernels:
  javascript    C:\Users\********\AppData\Roaming\jupyter\kernels\javascript
  python3       c:\users\********\documents\git\javascript\jupyter\.venv\share\jupyter\kernels\python3
```

試しに `jupyter kernelspec remove python3` で python3 のパスを削除してみる。

```bash
$ jupyter kernelspec list
Available kernels:
  python3       c:\users\********\documents\git\javascript\jupyter\.venv\lib\site-packages\ipykernel\resources
  javascript    C:\Users\********\AppData\Roaming\jupyter\kernels\javascript
```

すると以下のパスに変化した。remove するとデフォルト(?)のパスが設定されるのだろうか！？
そして相変わらずエラーは解消しない。

```bash
$ jupyter kernelspec list
Available kernels:
  javascript    C:\Users\********\AppData\Roaming\jupyter\kernels\javascript
  python3       c:\users\********\appdata\local\programs\python\python36-32\share\jupyter\kernels\python3
```

もしかしたら Python モジュールも Node モジュールも仮想環境だったり、プロジェクトローカルにインストールしていたのが悪かったのかもしれない。

なのでグローバルインストールしてみる。

```bash
npm install -g ijavascript
```

すると…**動いた**。これが原因か？

しかし `jupyter kernelspec list` のパスは最後の状態のままだ。python3 のパスが `c:\users\********\documents\git\javascript\jupyter\.venv\share\jupyter\kernels\python3` から `c:\users\********\appdata\local\programs\python\python36-32\share\jupyter\kernels\python3` に変わったことは何か影響あるのだろうか？

ただ `jupyter kernelspec install` などで元の状態への復旧を試みたが、どうしても戻らなかったのでこのまま動かすことにする。

## Node.js の少し困った(?)挙動

Jupyter Notebook は一つのノートブックが一つの REPL プロセスのような挙動を取る。つまりあるセルで定義した変数は保持され別のセルで参照可能だし、全てのセルを一旦削除して、新たにセルを作り直したとしても変数は保持されたままになり参照できる。

以下は Python のノートブックだが、このようにセルを跨いで変数の参照ができるし

![次のセルでも参照可能](/images/37-02.png)

以下のように一旦セルを全て削除しても変数は参照可能だ。

![一旦削除しても変数は残る](/images/37-03.png)

これは Node.js でも全く同じように動作するのだが、`const` を使った変数を定義すると少し困ったこと(?)になる。

Node.js のノートブックでも以下のように変数を定義してみよう。

![最初は問題ないように見えるが...](/images/37-04.png)

これは問題ない。しかし一旦書き直したくなってセルを全削除した場合はどうなるだろう

![二度目はエラーになる](/images/37-05.png)

このようにエラーになってしまう。つまり 2 度目からは `test` という変数が定義できず、書き直しができないのだ。

前述の挙動を理解ていれば当たり前に思えるのだが、最初どうなっているか分からず詰まってしまった。

対策としては `const` ではなく `var` や `let` を使えば良いし、何なら省略してもいい。ただノートブックでそこまでスコープを意識しないにしても、意図しない挙動を避けたいのであれば `let` を使うのが安心だろう。

```js
value = 1 // グローバルスコープ
var value2 = 1 // 関数スコープ
let value3 = 1 // ブロックスコープ
const value4 = 1 // ブロックスコープ
```

とはいえチョロっと書くようなノートは宣言なしでやってしまってもいいだろう。スコープの問題は頭の隅にでも留めておけば問題ない。

# Github でも見れる

Github に `.ipynb` ファイルをアップすると Jupyter Notebook としてレンダリングされる。

<a href="https://github.com/t-kojima/practice-black-python/blob/master/notebook/ghost_method.ipynb" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

イイネ！

# 参考

- [JavaScript の始め方 (Windows, Jupyter notebook) - YKpages](http://kato-robotics.hatenablog.com/entry/2018/03/27/155434)
- [n-riesco/ijavascript: IJavascript is a javascript kernel for the Jupyter notebook](https://github.com/n-riesco/ijavascript)
- [PENGUINITIS - Jupyter Notebook の Kernel error メモ](http://www.geocities.jp/penguinitis2002/computer/programming/Python/jupyter_notebook_kernel_error.html)
