# vrealloc {#vrealloc}

> vmallocによる割り当てのサイズを変更する

## vreallocとは何か {#what-is-vrealloc}

`vrealloc()` は既存のvmallocによる割り当てのサイズを変更する。ユーザー空間の `realloc()` に似ている。

```c
void *vrealloc(const void *p, size_t size, gfp_t flags);
```

## 歴史と変遷 {#history-evolution}

### v6.12: 最初の実装 {#v612-initial-implementation}

**コミット**: [3ddc2fefe6f3](https://git.kernel.org/linus/3ddc2fefe6f3) ("mm: vmalloc: implement vrealloc()")
**カーネル**: v6.12
**作者**: Danilo Krummrich

**なぜ必要だったか**:

既存の `kvrealloc()` は、`krealloc()` と比べて振る舞いに一貫性がなかった。

| 振る舞い | krealloc() | kvrealloc()（旧） |
|----------|------------|-------------------|
| size = 0 | メモリを解放する | 既存のポインタを返す |
| NULLポインタ | kmallocのように振る舞う | フォールトする |
| 旧サイズの追跡 | 自己完結している | 呼び出し側が追跡しなければならない |

**LKMLでの議論**: [Align kvrealloc() with krealloc()](https://lore.kernel.org/lkml/20240704170738.3621-1-dakr@redhat.com/)

**Rustからの動機**: このパッチはRustのアロケータ抽象化に言及している。`Vec` や `KVec` は拡大・縮小のために効率的なreallocを必要とする。これはよくあるパターンだが、適切なvreallocなしでは非効率だった。

### バグ: KASANのポイズニング（v6.12） {#bug-kasan-poisoning-v612}

**コミット**: [d699440f58ce](https://git.kernel.org/linus/d699440f58ce) ("mm: fix vrealloc()'s KASAN poisoning logic")
**報告者**: Andrii Nakryiko（BPFのテスト経由）
**バグ報告**: [lore.kernel.org](https://lore.kernel.org/bpf/67450f9b.050a0220.21d33d.0004.GAE@google.com/)

**何が起きたか**: vrealloc()が既存のvmap_areaを再利用するとき、KASANのアノテーションが新しいサイズに合わせて正しく更新されず、偽陽性のKASANレポートが発生した。

**教訓**: メモリサニタイザ（KASAN）は割り当てのコードパスを複雑にする。メモリ領域を再利用するときは、すべてのアノテーションを新しい有効/無効バイト範囲に合わせて更新しなければならない。

### v6.13: requested_sizeの追跡 {#v613-requested_size-tracking}

**コミット**: [a0309faf1cb0](https://git.kernel.org/linus/a0309faf1cb0) ("mm: vmalloc: support more granular vrealloc() sizing")
**作者**: Kees Cook

`vm_struct::requested_size` を導入し、実際に要求されたサイズを割り当て領域とは別に追跡するようにした。これにより次が可能になった。

- 正しいKASANポイズニング境界
- 既存の割り当て内での拡大
- 縮小最適化の基盤

**これを引き起こしたバグ報告**: [lore.kernel.org](https://lore.kernel.org/all/20250408192503.6149a816@outsider.home/)

### バグ: インプレース領域が返されない（v6.13） {#bug-in-place-region-not-returned-v613}

**コミット**: [f7a35a3c36d1](https://git.kernel.org/linus/f7a35a3c36d1) ("mm: vmalloc: actually use the in-place vrealloc region")
**報告者**: Shung-Hsi Yu（SUSE）
**バグ報告**: [lore.kernel.org](https://lore.kernel.org/all/20250515-bpf-verifier-slowdown-vwo2meju4cgp2su5ckj@6gi6ssxbnfqg)

**何が起きたか**: インプレースでの拡大をサポートするためのリファクタリングは、縮小の場合にしか機能していなかった。拡大する割り当てでは、処理は行うものの、結局まったく新しい領域を割り当てていた。

**影響**: BPF検証器（verifier）（vreallocのヘビーユーザー）における性能リグレッション。

**教訓**: 制御フローにおけるoff-by-one的なバグ。関数は正しい処理をすべて行っていたが、return文が誤った変数を使っていた。1行の修正で大きな影響があった。

### バグ: 拡大時の不要なゼロ初期化（v6.13） {#bug-unnecessary-zero-init-on-grow-v613}

**コミット**: [70d1eb031a68](https://git.kernel.org/linus/70d1eb031a68) ("mm: vmalloc: only zero-init on vrealloc shrink")

**何が起きたか**: `init_on_alloc` が有効な場合、コードは拡大時と縮小時の両方でメモリをゼロクリアしていた。しかし拡大時には、新しいメモリはアロケータによってすでにゼロクリアされていた。

**教訓**: 初期化が上流ですでに行われているのはいつかを理解すること。冗長なゼロクリアは大規模では高コストである。

### v6.14+: 縮小の最適化 {#v614-shrink-optimization}

**コミット**: [e64f42036ef4](https://git.kernel.org/linus/e64f42036ef4) ("mm/vmalloc: free pages when vrealloc() shrinks allocation")

**特定された問題**: `vrealloc()` による縮小はメタデータを更新するだけで、すべてのpageをマップしたままにしていた。`256KB` を `16KB` に縮小しても、依然として `256KB` の物理メモリを消費していた。

**修正内容**:

- `vunmap_range()` で新しいサイズを超えるpageのマップを解除する
- `__free_page()` でpageを解放する
- memcgのアカウンティングを更新する
- `nr_vmalloc_pages` カウンタを更新する

**結果**: `256KB` -> `16KB` に縮小したとき、メモリが93%削減された。

## 振る舞い {#behavior}

### 縮小 {#shrinking}

縮小時（size < 現在のサイズ）:

- 新しいサイズまでのデータを保持する
- 使われなくなったpageを解放する（e64f42036ef4以降）
- 同じポインタを返す（コピーなし）

### 拡大 {#growing}

拡大時（size > 現在のサイズ）:

- 余裕があれば既存の割り当てを再利用することがある（a0309faf1cb0以降）
- そうでなければ: 新たに割り当て、コピーし、古いものを解放する
- 新しいポインタを返すことがある

## 設計判断 {#design-decisions}

### なぜ縮小時にヒュージページはスキップされるのか {#why-huge-pages-are-skipped-for-shrink}

ヒュージページ（order > 0）は、複数のpageを覆う単一のTLBエントリを使う。部分的なマップ解除にはヒュージページの分割が必要になり、これは複雑なうえ、ヒュージページを使う意味を失わせる。ヒュージページの割り当てを縮小する場合は、メタデータだけを更新する。

### なぜ少なくとも1ページは残さなければならないのか {#why-at-least-one-page-must-remain}

先頭のpageは `mod_memcg_page_state()` を介したmemcgのアカウンティングに使われる。memcgのチャージは先頭のpageに関連付けられている。すべてのpageを解放すると、このアカウンティングの参照が失われる。

### なぜ縮小に最小しきい値がないのか {#why-no-minimum-threshold-for-shrink}

初期の版では、実際に解放する前に、解放対象が4ページ以上あることを要件とすることが検討された。これは削除された。

- 振る舞いは、恣意的なカーネルのポリシーではなく呼び出し側が制御すべきである
- 解放されたメモリはどんな量でも価値がある
- マジックナンバー的なしきい値はレビューで正当化しにくい

## よくある落とし穴 {#common-pitfalls}

vreallocの開発で見つかったバグに基づく:

1. **KASANのアノテーション**: メモリを再利用するときは、サニタイザの状態を更新する
2. **戻り値のバグ**: 複雑な操作の後、正しいものを返しているか確認する
3. **冗長な初期化**: 上流のコードがすでに何を初期化したかを把握する
4. **ヒュージページに対する部分的な操作**: 通常、分割しなければ不可能である

## テスト {#testing}

`test_vmalloc` モジュールには `vrealloc_shrink_test`（id: 4096）が含まれている。

```bash
# virtme-ng を使って QEMU で実行する
virtme-ng --append 'test_vmalloc.run_test_mask=4096'
```

## 参照 {#references}

### コミット（時系列順） {#commits-chronological}
| コミット | カーネル | 説明 |
|--------|--------|-------------|
| [3ddc2fefe6f3](https://git.kernel.org/linus/3ddc2fefe6f3) | [v6.12](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.12) | 最初の実装 |
| [d699440f58ce](https://git.kernel.org/linus/d699440f58ce) | [v6.12](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.12) | KASANポイズニングの修正 |
| [a0309faf1cb0](https://git.kernel.org/linus/a0309faf1cb0) | [v6.13](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.13) | requested_sizeの追跡 |
| [f7a35a3c36d1](https://git.kernel.org/linus/f7a35a3c36d1) | [v6.13](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.13) | インプレース領域を返す修正 |
| [70d1eb031a68](https://git.kernel.org/linus/70d1eb031a68) | [v6.13](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tag/?h=v6.13) | 冗長なゼロ初期化の修正 |
| [e64f42036ef4](https://git.kernel.org/linus/e64f42036ef4) | v6.14+ | 縮小の最適化 |

### LKMLでの議論 {#lkml-discussions}
- [最初のvrealloc提案](https://lore.kernel.org/lkml/20240704170738.3621-1-dakr@redhat.com/)
- [KASANのバグ報告](https://lore.kernel.org/bpf/67450f9b.050a0220.21d33d.0004.GAE@google.com/)
- [BPF検証器の速度低下](https://lore.kernel.org/all/20250515-bpf-verifier-slowdown-vwo2meju4cgp2su5ckj@6gi6ssxbnfqg)

### コード {#code}
- [`mm/vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) - 実装
- [`lib/test_vmalloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/test_vmalloc.c) - テストモジュール

### 関連 {#related}
- [vmalloc](vmalloc.md) - 親となるアロケータ

## 参考文献 {#further-reading}

- [vmalloc.md](vmalloc.md) — 親となるアロケータ。vmap_areaのRBTree、lazy TLBフラッシュ、ヒュージページによる裏付けを扱う
- [slab.md](slab.md) — `krealloc()` はslab/kmallocによる割り当てに対する同様のサイズ変更プリミティブである
- [kasan.md](kasan.md) — [d699440f58ce](https://git.kernel.org/linus/d699440f58ce) で修正されたv6.12のバグの原因となった、KASANポイズニングとの相互作用
- [mm/vmalloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmalloc.c) — `vrealloc()` の実装。`vrealloc` と `vm_struct::requested_size` を検索するとよい
- [lib/test_vmalloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/lib/test_vmalloc.c) — `vrealloc_shrink_test`（テストマスクid 4096）が拡大・縮小・インプレース再利用の各パスを検証する
- [3ddc2fefe6f3](https://git.kernel.org/linus/3ddc2fefe6f3) — v6.12でのDanilo Krummrichによる最初の `vrealloc()` 実装
- [a0309faf1cb0](https://git.kernel.org/linus/a0309faf1cb0) — Kees Cookによるv6.13での `vm_struct::requested_size` の追加。インプレースでの拡大を可能にした
- [e64f42036ef4](https://git.kernel.org/linus/e64f42036ef4) — サイズ縮小時に実際に物理pageを解放する縮小最適化
