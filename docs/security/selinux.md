# SELinux: 強制アクセス制御 {#selinux-mandatory-access-control}

> セキュリティコンテキスト、AVCキャッシュ、Type Enforcementポリシー、audit2allow

## SELinuxとは {#what-is-selinux}

SELinux（Security-Enhanced Linux）は**強制アクセス制御（MAC）**を実装する。アクセスの判定はオブジェクトの所有者ではなく、中央のセキュリティポリシーによって行われる。rootでさえ、明示的な許可なしにはポリシーを上書きできない。

```
Traditional DAC (Discretionary Access Control):
  owner of file → decides who can access it
  root → overrides everything

SELinux MAC:
  Central policy → decides every access
  root → still restricted by policy
  Even compromised root process → bounded by its security context
```

SELinuxはNSAが開発し、2000年12月にオープンソース化され、Linux 2.6.0-test3（2003年8月）にマージされた。**LSM（Linux Security Module）**フレームワークの上に構築されている。セキュリティ上重要なカーネル操作はすべてLSMフックを呼び出し、SELinuxがそれを横取りする。

## セキュリティコンテキスト {#security-contexts}

すべてのプロセスとファイルは**セキュリティコンテキスト**（ラベルとも呼ぶ）を持つ:

```
Format: user:role:type:level
         ↑     ↑    ↑    ↑
         SELinux user  sensitivity:categories (MLS/MCS)

Examples:
  Process: system_u:system_r:httpd_t:s0
  File:    system_u:object_r:httpd_var_run_t:s0
  Port 80: system_u:object_r:http_port_t:s0
```

```bash
# プロセスのコンテキストを表示する:
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0     1234 httpd

# ファイルのコンテキストを表示する:
ls -Z /var/www/html/index.html
# system_u:object_r:httpd_content_t:s0 /var/www/html/index.html

# 現在の自分のコンテキスト:
id -Z
# unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

# 一時的にコンテキストを変更する:
runcon -t httpd_t /bin/bash    # httpd_tとしてシェルを実行する

# すべてのコンテキストを表示する:
seinfo -t | grep httpd    # すべてのhttpdタイプ
```

## Access Vector Cache（AVC） {#access-vector-cache-avc}

すべてのアクセスチェックは**AVC**（Access Vector Cache）を通る。AVCは最近の許可/拒否の判定をキャッシュするハッシュテーブルである:

```c
/* security/selinux/avc.c */

struct avc_entry {
    u32             ssid;       /* ソースのセキュリティID */
    u32             tsid;       /* ターゲットのセキュリティID */
    u16             tclass;     /* ターゲットのクラス（file、process、socket……） */
    struct av_decision avd;     /* allow/auditなどのビットマスク */
};

/* キャッシュの検索: */
static inline int avc_lookup(struct selinux_avc *avc,
                              u32 ssid, u32 tsid, u16 tclass,
                              struct av_decision *avd)
{
    struct avc_node *node;
    unsigned long flags;

    node = avc_cache_lookup(avc, ssid, tsid, tclass);
    if (node) {
        *avd = node->ae.avd;
        /* キャッシュヒット: ポリシーを参照する必要はない */
        return 0;
    }
    /* キャッシュミス: 低速パス → ポリシーデータベースを参照する */
    return avc_compute_av(ssid, tsid, tclass, avd);
}
```

```bash
# AVCの統計:
cat /sys/fs/selinux/avc/cache_stats
# lookups hits misses allocations reclaims frees
# 1234567 1200000 34567 34567 0 0
# 典型的なヒット率は約97%

# AVCキャッシュのサイズ:
cat /sys/fs/selinux/avc/cache_threshold
# 512（エントリ）
```

## Type Enforcement（TE）ポリシー {#type-enforcement-te-policy}

最も重要なポリシーの仕組みである。ルールはタイプ間のアクセスを許可する:

```
allow <source_type> <target_type>:<class> { <permissions> };

Examples:
# httpdはhttpd_content_tのファイルを読める:
allow httpd_t httpd_content_t:file { read getattr open };

# httpdは/etc（etc_t）に書き込めない:
# （allowルールがない → 拒否）

# httpdはhttp_port_tのポートにbindできる:
allow httpd_t http_port_t:tcp_socket { name_bind };

# httpdにmysqlへの接続を許可する:
allow httpd_t mysqld_port_t:tcp_socket name_connect;
```

### ポリシーのソースファイル {#policy-source-files}

```bash
# ポリシーは.te（type enforcement）ファイルからコンパイルされる:
ls /usr/share/selinux/targeted/*.pp    # コンパイル済みのポリシーモジュール

# httpdのポリシーを表示する:
sesearch --allow -s httpd_t /etc/selinux/targeted/policy/policy.33
# allow httpd_t httpd_config_t:file { read getattr open };
# allow httpd_t httpd_log_t:file { append create write ioctl lock };
# ...

# あるタイプに何ができるかを確認する:
sesearch --allow -s httpd_t -c file /etc/selinux/targeted/policy/policy.33

# あるファイルタイプにどのタイプがアクセスできるか:
sesearch --allow -t httpd_content_t /etc/selinux/targeted/policy/policy.33
```

## ファイルコンテキストのラベル付け {#file-context-labeling}

```bash
# デフォルトのコンテキストはfile_contextsファイルから来る:
cat /etc/selinux/targeted/contexts/files/file_contexts | grep www
# /var/www(/.*)?       system_u:object_r:httpd_content_t:s0
# /var/www/cgi-bin(/.*)? system_u:object_r:httpd_exec_t:s0

# コンテキストをポリシーのデフォルトに戻す:
restorecon -Rv /var/www/html/

# コンテキストを手動で設定する:
chcon -t httpd_content_t /tmp/webpage.html
semanage fcontext -a -t httpd_content_t "/custom/web(/.*)?"
restorecon -Rv /custom/web/

# ポリシー上、ファイルのコンテキストがどうあるべきかを確認する:
matchpathcon /var/www/html/index.html
# /var/www/html/index.html system_u:object_r:httpd_content_t:s0
```

## auditとaudit2allow {#audit-and-audit2allow}

SELinuxはアクセスを拒否すると、auditログに記録する:

```bash
# 最近の拒否を表示する:
audit2allow -w -a 2>/dev/null | head -30
# type=AVC msg=audit(1234567890.123:456): avc: denied { name_connect }
#     for pid=1234 comm="httpd" dest=3306
#     scontext=system_u:system_r:httpd_t:s0
#     tcontext=system_u:object_r:mysqld_port_t:s0
#     tclass=tcp_socket permissive=0

# 生のauditログ:
ausearch -m AVC,USER_AVC -ts recent | audit2allow

# 拒否を許可するポリシーを生成する:
audit2allow -a -M myhttpd    # myhttpd.ppモジュールを作成する
semodule -i myhttpd.pp       # モジュールをインストールする

# あるいは追加すべきものを対話的に提案させる:
audit2allow -a
# allow httpd_t mysqld_port_t:tcp_socket name_connect;
```

## booleanスイッチ {#boolean-switches}

booleanにより、再コンパイルせずにポリシーを調整できる:

```bash
# すべてのbooleanを一覧表示する:
getsebool -a | grep httpd | head -10
# httpd_can_network_connect --> off
# httpd_can_network_connect_db --> off
# httpd_can_send_mail --> off
# httpd_enable_cgi --> on

# httpd → データベースの接続を有効にする:
setsebool -P httpd_can_network_connect_db on
# -P = 永続的（再起動後も維持される）

# Webアプリケーションでよく使うboolean:
setsebool -P httpd_unified on          # httpdに任意のhttpdタイプの読み取りを許可する
setsebool -P httpd_enable_homedirs on  # ~user/public_htmlを許可する
```

## SELinuxのモード {#selinux-modes}

```bash
# 現在のモードを確認する:
getenforce
# Enforcing

# モード:
# Enforcing:  ポリシーを適用し、違反は拒否 + 監査される
# Permissive: ポリシーは適用されないが、違反は監査される（AVCの拒否が記録される）
# Disabled:   SELinuxは完全にオフ（再び有効にするには再起動が必要）

# 一時的にpermissiveに切り替える（デバッグ用）:
setenforce 0   # permissive
setenforce 1   # enforcing

# ドメインごとのpermissive（そのプロセスタイプはpermissive、他はenforcing）:
semanage permissive -a httpd_t   # httpd_tをpermissiveモードにする
semanage permissive -d httpd_t   # permissiveを解除する
```

## selinuxfs: カーネルインターフェース {#selinuxfs-kernel-interface}

```bash
# SELinuxはselinuxfs経由でインターフェースを公開する:
ls /sys/fs/selinux/
# access  avc  booleans  class  commit_pending_bools  context
# create  deny_unknown  disable  enforce  load  mls  null  policy
# policyvers  relabel  status  user

# SELinuxが有効か確認する:
cat /sys/fs/selinux/enforce
# 1 = enforcing

# 新しいポリシーをロードする:
cat /etc/selinux/targeted/policy/policy.33 > /sys/fs/selinux/load

# ポリシーのバージョンを確認する:
cat /sys/fs/selinux/policyvers
# 33
```

## カーネルの実装 {#kernel-implementation}

```c
/* security/selinux/hooks.c */

/* SELinuxはセキュリティ上重要なすべてのVFS操作にフックする: */
static int selinux_inode_permission(struct inode *inode, int mask)
{
    const struct cred *cred = current_cred();
    u32 perms;
    int rc;

    /* 現在のプロセスと対象inodeのセキュリティIDを取得する */
    u32 ssid = selinux_cred(cred)->sid;
    u32 tsid = selinux_inode(inode)->sid;
    u16 sclass = inode_mode_to_security_class(inode->i_mode);

    perms = file_mask_to_av(inode->i_mode, mask);

    /* AVCチェック: ssidはtsid:sclassに対してpermsを許可されているか */
    rc = avc_has_perm(ssid, tsid, sclass, perms, &ad);
    return rc;  /* 0 = 許可、-EACCES = 拒否 */
}
```

## 参考文献 {#further-reading}

- [LSMフレームワーク](lsm.md) — SELinuxが使うセキュリティフック基盤
- [ケーパビリティ](capabilities.md) — SELinuxとケーパビリティの相互作用
- [Linuxのaudit](audit.md) — SELinuxの拒否はauditログに送られる
- [クレデンシャル](credentials.md) — クレデンシャルから導出されるプロセスのSID
- `security/selinux/` — SELinuxの実装
- `man semanage, restorecon, audit2allow` — ポリシー管理ツール
- Red Hat SELinux Guide — ポリシー記述に関する包括的なドキュメント
