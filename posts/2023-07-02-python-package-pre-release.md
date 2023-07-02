# Python 패키지 프리릴리즈 버전관리로 의존성 관리하기

일반적인 GitHub을 통한 버전 관리 시스템에서는 아래와 같이 MAJOR.MINOR.PATCH 형식으로 버전관리를 권장하며

해당 브랜치에 대한 최신 커밋을 대상으로 태깅을 진행합니다.

<img width="289" alt="image" src="https://github.com/jongwony/pages/assets/12846075/0cbaf48e-9abd-48bf-b163-86bb201c0dc6">

Python Package 버전 관리는 비슷하지만 다릅니다.

바로 [PEP-440](https://peps.python.org/pep-0440/) 을 통해서 규약이 정해져 있습니다.

GitHub 와는 조금 다르게, 태깅을 하면 그제서야 패키지가 생기는 방식이 아닌 dist 패키지를 업로드 하자마자 정해지기 때문에, 

<img width="729" alt="image" src="https://github.com/jongwony/pages/assets/12846075/4716c035-e984-46ca-a856-d00f57438cba">

실제 [Python GitHub](https://github.com/python/cpython) 에서 Network Graph 를 확인해보시면 main 과 minor 까지만 브랜치 관리가 되고 있음을 알 수 있고,
minor 브랜치 위에 프리릴리즈가 같이 올라가는것을 확인할 수 있었습니다.

자 그럼 pip 패키지를 사내에서 private 방식으로 운영하고 있는 입장에서는 Git Flow 를 따르는게 맞나? 하는 생각이 듭니다.

아래 도식을 보시죠.

![image](https://github.com/jongwony/pages/assets/12846075/188553ce-6992-41a2-9905-29aec0a32a0e)

Stage 영역에는 Stage 패키지를 쓰고, Production 영역에는 Production 패키지를 쓰는 그림입니다.

이렇게 하면 사실상 다른 패키지를 사용하는것과 마찬가지며 Stage 가 Production 에 머지 되는 플로우를 만들기 어렵습니다.

휴먼 에러의 여지가 굉장한 플로우겠네요

Python PEP-440 에서는 의존되는 패키지의 pre release 개념을 사용하도록 하여 브랜치 하나로 관리하는게 자연스러운것으로 보입니다.

![image](https://github.com/jongwony/pages/assets/12846075/88c9c88f-a88f-49cd-96e7-45bae53981e0)

이러면 자동화가 가능합니다.

Stage 영역에는 아래와 같이 Final Release 를 먼저 설치한 후 Pre Release 를 no-deps 옵션으로 추가 설치합니다.

```
pip install --upgrade private_package
pip install --pre --no-deps --upgrade private_package
```

Production 영역에는 아래와 같이 Final Release 만 설치합니다.


```
pip install --upgrade private_package
```

pypi server 에서는 aN(알파), bN(베타), rcN(릴리즈 후보) 버전을 올리면 스테이징으로 자동 배포되고
N.N(.N) 버전을 배포 함으로써 final release 버전을 올리면 스테이징 및 프로덕션에 자동 배포되는 전략을 취할 수 있습니다.

의존성 패키지는 final release 를 사용하고 코드 변경만 private 패키지에 의존하는 깔끔한 전략을 가져갈 수 있습니다.
