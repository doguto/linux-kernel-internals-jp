# Linuxのセキュリティ {#linux-security}

> 権限モデル、強制アクセス制御、syscallフィルタリング

## セキュリティの階層 {#security-layers}

Linuxは階層化されたセキュリティモデルを採用している:

```
Application
    │
    ▼
Syscall entry
    │
    ├── DAC (Discretionary Access Control): uid/gid/permissions ─── always
    │
    ├── Capabilities: fine-grained privilege splitting ──────────── always
    │
    ├── LSM hooks (SELinux / AppArmor / TOMOYO) ──────────────────── if enabled
    │
    └── seccomp BPF: syscall whitelist/blacklist ─────────────────── if installed
```

**DAC**（ファイルパーミッション、uid/gid）は常に適用される。**LSM**はその上に強制アクセス制御（Mandatory Access Control）ポリシーを追加する。**ケーパビリティ**は「root権限」を40個の個別の権限に分割する。**seccomp**はプロセスが発行できるsyscallを制限する。

## このセクションのページ {#pages-in-this-section}

### アクセス制御 {#access-control}

| ページ | 扱う内容 |
|------|----------------|
| [LSMフレームワーク](lsm.md) | LSMフック、SELinux、AppArmorのアーキテクチャ |
| [SELinux](selinux.md) | ラベル、ポリシー、AVC — MACの判定がどのように行われるか |
| [Landlock](landlock.md) | ユーザー空間からの非特権サンドボックス化 |
| [ケーパビリティ](capabilities.md) | Linuxのケーパビリティモデル、権限の放棄 |
| [クレデンシャル](credentials.md) | struct cred、uid/euid/fsuid、クレデンシャルのライフサイクル |
| [ユーザーnamespace](user-namespaces.md) | 非特権コンテナとその攻撃対象領域 |

### syscallとカーネルのハードニング {#syscall-kernel-hardening}

| ページ | 扱う内容 |
|------|----------------|
| [seccomp BPF](seccomp.md) | syscallフィルタリング、libseccomp、コンテナ用プロファイル |
| [カーネルのハードニング](kernel-hardening.md) | スタックプロテクタ、CFI、緩和策の全体像 |
| [Audit](audit.md) | auditサブシステム、ルール、そのコスト |

### ストレージ暗号化 {#storage-encryption}

| ページ | 扱う内容 |
|------|----------------|
| [fscrypt](fscrypt.md) | ファイルシステムレベルの暗号化（ext4/f2fs） |
| [dm-crypt](dm-crypt.md) | ブロックレベルの暗号化、LUKS |

### 事件簿 {#war-stories}

| ページ | 扱う内容 |
|------|----------------|
| [セキュリティの事件簿](war-stories.md) | 実在する4つのCVE — Dirty Pipe、fs_contextのオーバーフロー、古いptracerクレデンシャル、ネストしたユーザーnamespaceのUID/GIDマッピング — の根本原因を分析し、それぞれについて実際に存在する公開記録をたどる |
