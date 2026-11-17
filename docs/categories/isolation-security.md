# 分離とセキュリティ {#isolation-security}

ワークロードを分離し、閉じ込め、保護する仕組み — コンテナからアクセス制御、ゲスト全体の実行まで。

- [Cgroups（cgroups/）](../cgroups/README.md) — プロセスのグループごとにリソースを計上・制限する。コンテナの基盤
- [セキュリティ（security/）](../security/README.md) — LSM、ケーパビリティ、seccomp、そしてカーネルのアクセス制御機構
- [仮想化（virtualization/）](../virtualization/README.md) — KVMと、カーネルがゲストマシンを実行する仕組み
- [暗号（crypto/）](../crypto/README.md) — カーネルの暗号APIとハードウェアアクセラレーション
- [Livepatch（livepatch/）](../livepatch/README.md) — 実行中のカーネルを再起動せずにパッチする
