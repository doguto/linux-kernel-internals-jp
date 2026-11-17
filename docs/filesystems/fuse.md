# FUSE: ユーザー空間のファイルシステム {#fuse-filesystems-in-userspace}

> カーネルが*普通のプログラム*にファイルシステムを実装させる仕組み — `/dev/fuse`上のリクエスト／レスポンスプロトコル、そのコストがなぜそうなるのか、そしてvirtiofsとio_uringがその計算をどう変えるのか

このセクションの他の内容はすべてカーネルコードである。ext4、XFS、btrfsはカーネルに組み込まれ、カーネルコンテキストで動作する。FUSEは例外であり、それ以外のすべてを回避する道を開く — 通常のユーザー空間プロセスがファイルシステムを実装できるようにするのだ。sshfs、s3fs、glusterのクライアント、`ntfs-3g`、GNOMEのgvfs、AppImage、rcloneのマウントはすべてこの方式で動作する。トレードオフは、柔軟性（任意の言語で、カーネルコードなし、再起動なしでファイルシステムを書ける）と、アーキテクチャから直接生じる性能コストとの間にある。

## アーキテクチャ: デーモンへ転送するカーネルドライバ {#the-architecture-a-kernel-driver-that-forwards-to-a-daemon}

FUSEはデバイスを通じて対話する2つの部分からなる。

1. 実際のファイルシステムタイプとして`fuse`を登録する**カーネルモジュール**（`fs/fuse/`）。VFSから見ると他のファイルシステムと変わらない — inode操作とファイル操作を実装している。
2. 実際のファイルシステムロジックを含む**ユーザー空間デーモン**（通常は`libfuse`上に構築される）。

両者はキャラクタデバイス**`/dev/fuse`**を通じて通信する。プロセスがFUSEマウント上で`open()`、`read()`、`stat()`、`readdir()`を実行すると、VFSはFUSEカーネルモジュールを呼び出す。しかしモジュールは答え方を*知らない*。代わりに操作を**リクエスト**にまとめ、デーモンが`/dev/fuse`から読み出せるようにする。デーモンはリクエストを読み、それが意味する処理（S3からの取得、SSH越しの読み出し、NTFSのデコード）を行い、**リプライ**を書き戻す。カーネルはリプライを展開し、ずっとブロックしていた元の呼び出し元へ返す。

```
  user process              kernel                         FUSE daemon (userspace)
  ────────────              ──────                         ───────────────────────
  read(fd) ──────────►  VFS ──► FUSE module
                                    │  build FUSE_READ request
                                    ▼
                               /dev/fuse   ────read()────►  handle it (e.g. HTTP GET)
                                    ▲                              │
                                    └────────write() reply ◄───────┘
  ◄──────── data ───── VFS ◄── FUSE module (copies reply into the page)
```

各リクエストはオペコード（`FUSE_LOOKUP`、`FUSE_OPEN`、`FUSE_READ`、`FUSE_WRITE`、`FUSE_GETATTR`、…）とその引数を運ぶ。プロトコルはバージョン管理されており、マウント時に`FUSE_INIT`ハンドシェイクでネゴシエートされる。

## コストがなぜそうなるのか {#why-it-costs-what-it-costs}

ネイティブなファイルシステムならページキャッシュのヒットで応答する1回の`read()`が、FUSEではユーザー空間への往復になる。

- **コンテキストスイッチ。** 呼び出し元タスクはブロックし、デーモンはリクエストを読むためにスケジュールされなければならず、その後に制御が戻ってくる。キャッシュされていない操作1回につき、少なくとも2つの余分なスケジューリング境界が生じる。
- **データコピー。** 歴史的に、リプライのデータはデーモンのバッファから`/dev/fuse`を経てカーネルのpageへコピーされていた。後に`splice()`がこれらのコピーの一部を取り除いたが、境界の横断は残っている。
- **シリアライズ。** すべての操作はFUSEプロトコル形式にマーシャリングされ、再びアンマーシャリングされる。

FUSEはこれを排除するのではなく緩和する。カーネル側は積極的にキャッシュする — dentry、属性、pageデータはネゴシエートされたタイムアウト付きでキャッシュされるため、繰り返しのルックアップやキャッシュ済みの読み出しはカーネルの外に出ない。ライトバックキャッシングはダーティデータをまとめる。`max_pages`とリードアヘッドにより、1つのリクエストで最大1メガバイトずつ移動できる。しかし「キャッシュミスのたびにユーザー空間へ渡る」という根本的なコストはモデルに固有のものであり、メタデータの多いワークロードや小さなI/Oのワークロードで、FUSEファイルシステムがネイティブなものに後れをとるのはこのためである。

## セキュリティ: 非特権マウント {#security-unprivileged-mounts}

FUSEデーモンは他のプロセスにファイルシステムの応答を返す信頼できないユーザー空間であるため、そのマウントは歴史的に特権操作だった。カーネルは**user namespace内での完全に非特権なFUSEマウント**のサポートを得た（[`4ad769f3c346`](https://git.kernel.org/linus/4ad769f3c346) "fuse: Allow fully unprivileged mounts"）。これにより、rootlessコンテナやサンドボックスがFUSEファイルシステムを安全にマウントできる — カーネルはそうしたマウントができることを制限し（例えば`allow_other`はマウントしたnamespaceの中に限定される）、悪意あるデーモンが自身のドメイン外のプロセスを攻撃できないようにしている。

## 古典的モデルを超えて {#beyond-the-classic-model}

FUSEの中心的なコスト — ユーザー空間への往復 — に挑む2つの進展がある。

- **virtiofs**は、FUSEプロトコルを別の問題、すなわちホストのディレクトリを仮想マシンと共有する問題に適用する。リクエストは`/dev/fuse`ではなく**virtio**トランスポートを通ってホストが処理するデバイスへ送られ、VMにネイティブに近いセマンティクスを持つ共有ファイルシステムを提供し、さらにDAXを介してホストのページキャッシュを直接マッピングする能力を与える。カーネルの[virtiofs documentation](https://docs.kernel.org/filesystems/virtiofs.html)を参照（[`a62a8ef9d97d`](https://git.kernel.org/linus/a62a8ef9d97d) "virtio-fs: add virtiofs filesystem"）。
- **FUSE over io_uring**は、`/dev/fuse`に対するread/writeのリクエストループを共有リングバッファに置き換える。これによりカーネルとデーモンははるかに少ないシステムコールとコンテキストスイッチでリクエストとリプライを交換でき、リクエストを同じCPU上で処理し続けることもできる。[FUSE-over-io-uring](https://docs.kernel.org/filesystems/fuse/fuse-io-uring.html)を参照。（リング機構そのものについては[io_uring](../io-uring/README.md)を参照。）

## 参考文献 {#further-reading}

- [Kernel docs: FUSE](https://docs.kernel.org/filesystems/fuse/fuse.html) — プロトコルと設計の根拠
- [Kernel docs: virtiofs](https://docs.kernel.org/filesystems/virtiofs.html) — VM向けにvirtio上で動くFUSEプロトコル
- [Kernel docs: FUSE over io_uring](https://docs.kernel.org/filesystems/fuse/fuse-io-uring.html) — リングベースのトランスポート
- [VFS](../vfs/README.md) — FUSEが単なる1つのファイルシステムタイプとして接続されるレイヤ
