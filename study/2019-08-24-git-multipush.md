date: 2019-08-24 17:32:00
title: git push multiple server
tags: ['git', 'ssh', 'multiple', 'aws', 'lightsail', 'remote']
layout: post

```
git push origin master
```

`origin` 에 여러 서버를 한번에 설정할 수 있습니다.

github 서버 외에 다른 ssh 서버에 직접 푸시하여 코드 배포를 간소화 하는 방법입니다.

여기서는 AWS Lightsail 을 예로 진행하겠습니다.

![Image](/static/images/git-multipush/2019-08-24-17:52.png)

```
ssh -i "LightsailDefaultKey-ap-northeast-2.pem" ec2-user@tech.jongwony.com
```

```
# home directory
git config --global credential.helper store
git clone https://github.com/jongwony/flask_blog.git
rm -rf flask_blog/.git

mkdir ssh_repo.git
cd ssh_repo.git
git init --bare

touch ssh_repo.git/hooks/post-receive
```

`post-receive`

```bash
#!/bin/sh
git --work-tree=$HOME/flask_blog --git-dir=$HOME/ssh_repo.git checkout -f
```

`--work-tree` 옵션이 코드가 관리되는 디렉터리고 `--git-dir` 가 `.git/` 에 대응되는 원격 디렉터리입니다.

```
git remote show origin
```

여기에서 `Push URL`을 아래 명령어로 두개를 추가할 수 있습니다.

```
git remote set-url --add --push origin https://github.com/jongwony/flask_blog.git
git remote set-url --add --push origin tech.jongwony.com:ssh_repo.git
```

하지만 `git remote` 명령어는 키 파일을 지정하는 옵션이 없기 때문에 설정으로 만들어줍시다.

```
# ~/.ssh/config
Host tech.jongwony.com
    HostName        tech.jongwony.com
    User            ec2-user
    IdentityFile    LightsailDefaultKey-ap-northeast-2.pem
```

이제 `git push origin master` 명령 하나로 github 에 코드가 배포됨과 동시에 Lightsail 서버에도 코드가 배포가 됩니다.

이는 static 페이지 업로드를 자주하는 블로그의 특징에 적절하며 코드 배포와 서버 배포를 분리할 수도 있습니다.

예를 들면 circle-ci 같은 걸 이 뒤에 붙이면 될 것 같네요.
