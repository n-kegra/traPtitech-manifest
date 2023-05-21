# manifest

ArgoCD参照用 k8s(k3s) manifestファイル群

## 書き始める前に

CI(Continuous Integration)でのyamlのバリデーションが無い場合、各自のエディタに以下のような拡張機能をインストールし、補完を頼ると良いでしょう

### VSCode

ref: [Kubernetesエンジニア向け開発ツール欲張りセット2022](https://zenn.dev/zoetro/articles/9454a6231a1273#vscode-extensions)

[YAML - Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=redhat.vscode-yaml) をインストールし、以下を `.vscode/settings.json` に追加

```json
{
    "yaml.schemas": {
        "kubernetes": [
            "*.yml",
            "*.yaml"
        ]
    }
}
```

CRD(Custom Resource Definition)の補完は知らない

### IntelliJ IDEA Ultimate

[Kubernetes - IntelliJ IDEs Plugin | Marketplace](https://plugins.jetbrains.com/plugin/10485-kubernetes)

`Languages & Frameworks > Kubernetes` より、CRD定義のURLを追加すると、CRDの補完も効くようになります
e.g. `https://raw.githubusercontent.com/argoproj/argo-cd/master/manifests/crds/application-crd.yaml`

## Secretの追加方法

Secretは別リポジトリで管理しています。

https://github.com/traPtitech/manifest-secret

Pushのタイミングによっては、manifestかmanifest-secretのsyncエラーが出ることがあるかもしれませんが、
typoでもしていなければ少し待つと正常になります。

## Bootstrap

万が一k8sのリソースが全部吹き飛んだ場合に1から構築する方法

まずは現在のリソースをチェック: `$ kubectl get --all-namespaces all`
ArgoCDの文字が見えなければ以下を行ってください

1. Ansibleを実行してk3sクラスタを構築
    - [SysAd/tokyotech.org: traP Infrastructure as Code - tokyotech.org - traP Git](https://git.trap.jp/SysAd/tokyotech.org)
    - master(k3s-server) → worker(k3s-agent) の順で実行することに注意
    - master(k3s-server) 実行後クラスタが再構築された場合、 `k3s_token` の値を書き換えるのを忘れないこと そうしないとworkerがjoinできません
2. ArgoCDをインストール
    - `kubectl create ns argocd`
    - `kubectl apply -n argocd -f {{ ./argocd/kustomization.yaml に書かれている install.yaml のURL }}`
    - `kubectl port-forward svc/argocd-server -n argocd 8124:443`
3. localhost:8124 にアクセス
    - sshしている場合はlocal forwardしてアクセス e.g. `ssh -L 8124:localhost:8124 remote-name`
    - Admin password: `kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo`
4. ArgoCDのUIから `applications` アプリケーションを登録
    - SSH鍵を手元で生成して、公開鍵をGitHubのこのリポジトリ (manifest) と manifest-secret に登録
    - 必要な場合は先にknown_hostsを登録 (GitHubのknown_hostsはデフォルトで入っている)
    - URLはSSH形式で、秘密鍵をUIで貼り付けてリポジトリを追加
    - アプリケーションを追加 (path: `applications`)
    - Syncを行う
5. cd.trap.jp にアクセスできるようになるはず
    - ArgoCDアプリケーションがsyncされた後はargocd serviceのポートは443番から80番になるので注意
    - local forwardでのアクセスを続けたい場合は `kubectl port-forward svc/argocd-server -n argocd 8124:80`
