---
title: "[Terraform][M1 Mac] CPUアーキテクチャミスマッチ"
date: "2022-05-11T11:17"
description: tfenvをdockerでおきかえました
---

こんにちは、fluoriteのfです。

仕事柄 Terraform を使うことが多いのですが、プロジェクトの性質上新しいバージョンに移行できないケースは多いです。
そして仕事で使うマシンを M1 Mac に変更することになったのですが、tfenv を使って古いバージョンのTerraformを使おうとするとハマるポイントがあります。


## tfenv とは

[tfenv](https://github.com/tfutils/tfenv)

.terraform-version にTerraformのバージョンを記述しておくと、terraformコマンドを実行するだけで、指定されたバージョンのTerraformに切りかえてくれたり、必要に応じてインストールしてくれる仕組です。
.terraform-vesionはテキストファイルなので、git commitして管理すると便利ですね。

## tfenv で古いバージョンを使おうとすると、ビルドされたバイナリが見つからないエラー

ローカルにインストールされていないバージョンのterraformがあると自動でインストールしてくれる仕組なのですが、TerraformのダウンロードサイトにApple Silicon用のバイナリがないバージョンがあります。

## amd64版をインストールすると動くという話

試してみたところ、3回に1回程度エラーになるようでした。

残念。

## そうだ、docker使おう

docker run を使えば、ローカルに存在していないコンテナイメージは自動的にダウンロードしてくれますし、そのまま実行してくれるので tfenv でやりたいことは既に備わっています。
あとは、AWSのクレデンシャルをプロセスに渡してあげること、terraformソースのマウント、pluginキャッシュなどを解決させるコマンドにすれば良さそうですね。

zshrc に SHELL関数を用意してみました。

```
# 親ディレクトリを辿って.git のあるディレクトリを出力する関数 
find_gitdir(){
  (
  until ls ./.git >/dev/null 2>&1
  do
    cd .. || break
  done
  pwd
  )
}

# terraform 関数。AWS_XXXX は aws-vault を使った際に設定される環境変数。
terraform() {
  version=$(cat .terraform-version)
  repo_root=$(find_gitdir)
  cwd=$(pwd)
  relative=${${cwd#$repo_root}:1}

  docker run -it --rm \
    --platform linux/amd64 \
    -v $repo_root:/repo \
    -w /repo/$relative \
    -e AWS_VAULT \
    -e AWS_DEFAULT_REGION \
    -e AWS_REGION \
    -e AWS_ACCESS_KEY_ID \
    -e AWS_SECRET_ACCESS_KEY \
    -e AWS_SESSION_TOKEN \
    -e AWS_SECURITY_TOKEN \
    -e AWS_SESSION_EXPIRATION \
    -v $HOME/.ssh:/root/.ssh \
    -v $HOME/.terraform.d:/root/.terraform.d \
    -v $HOME/.terraformrc:/root/.terraformrc \
    hashicorp/terraform:"$version" "$@"
}
```
gitリポジトリ前提でリポジトリのルートディレクトリごとマウントしました。
実際にterraformを実行したいパスはルートディレクトリではないケースが多いので、workdirとして指定しています。
.terraformrc にplugin-cacheの設定があります。そのため、.terraformrcと .terraform.d をマウントしました。
terraformのソース内で、EC2インスタンスにSSHする処理がある場合に供えて .sshディレクトリもマウントしました。あったとしてもレアケースだと思いますね。やりすぎかもしれません。
--platform が linux/amd64 なんですよね。これは Rancher Desktopを使っているからではないかと思っています。linux/arm じゃない理由は今後探っていこうと思います。
- 