# AWS CLI Windows 실행이 안되는 오류 해결 과정

최근 인스턴스의 액세스 키를 교체하는 과정에서 크리덴셜을 제대로 인식하지 못하는 경우가 발생했다.

기존 인스턴스 내에 `aws configure` 로 지정한 키를 사용하던 방식을 제거하고 DevOps 에서 인스턴스에 SSM, CloudWatch Role 이 부착을 해서 제공했기 때문에

`~/.aws`, AWS CLI 를 제거한 것이 모든 원흉의 시작이었다.

모든 것을 무(無)로 되돌려 놨더니 과거 인스턴스에 설치 안 된 에이전트가 많았다는 것을 뒤늦게 알게 된 과정 역시 공유한다.

일단 설치한 건 AWS CLI V2 를 새로 설치하였고 일단 Windows 의 CLI 사용을 위한 환경변수를 지정해야 했다.
[msi 로 설치](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-cliv2-windows.html)하면 자동으로 환경변수도 지정해준다

이렇게 기본적으로 가리키는 AWS 서비스가 설치되는 Windows 의 경로는 `C:\Program Files\Amazon` 경로였다

이게 설치만 해서 크리덴셜을 인식해야 하는데(인스턴스에 IAM 롤을 부여했으니!), 인식하지 못한다

```
botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

왜!?

삽질을 좀 하고, 리서치도 좀 하다가 아래와 같은 Windows 아키텍처를 발견했다.

![image](https://user-images.githubusercontent.com/12846075/132823331-e446c946-55ba-45af-8826-52e6a098e0b3.png)

저 그림에 따르면 CLI 여부에 상관없이 일단 169.254.169.254 주소가 접속이 되어야 한다는 의미라서 접속을 해봤다

![image](https://user-images.githubusercontent.com/12846075/132823685-8a42ff4e-923a-4fb1-87fb-e94e7873a276.png)

파이썬으로 했음

그럼 이게 문제니 뭔가 라우팅이 잘못되었다는 이야기구나 한번 `route print` 라는 파워셸의 툴을 사용해서 조회해봤다.

![image](https://user-images.githubusercontent.com/12846075/132824404-34f2b961-8511-45a2-a133-7ef73f2572e6.png)

Active Routes 안에 169.254.169.254 가 전혀 없다!

이걸 어떻게 해결하지 하다가

https://www.youtube.com/watch?v=DEkrcNkF-4Q 요 영상을 찾게 되어 비슷한 시도를 했음을 알았다

근데 중간에 보니 얼마나 옛날 인스턴스면은

![image](https://user-images.githubusercontent.com/12846075/132824829-f9458bdc-8d35-4b71-8d6c-0d6ae6239f17.png)

여기 원래 EC2Launch 모듈도 없었다 그래서 깔았다

https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/WindowsGuide/ec2launch-v2-install.html

최신버전 EC2Launch는 명령어도 다르다(예전 버전은 영상을 따라하면 될 듯함) 아래와 같이 help로 명령어를 한참 찾다가 `EC2Launch.exe run Add-Routes` 를 쳐봤더니 해결되었다.

![image](https://user-images.githubusercontent.com/12846075/132824950-3bceba6b-b9cd-45f0-902a-462df2cf8a5e.png)

![image](https://user-images.githubusercontent.com/12846075/132825137-32276770-c978-4647-8d45-19ade86eb9d0.png)

Active Routes 항목에 169.254.169.254 가 추가되었고 정상적으로 Role 에 달려있는 권한을 `aws configure` 없이 가져올 수 있게 되었다!

## 요약

- aws cli default 프로필이 없으면 169.254.169.254 에서 임시 세션을 얻기 때문에 인스턴스에 롤을 붙이기만 하면 되는 것이었음.
- 169.254.169.254 가 안되면 큰일이니 라우팅을 확인해보자.
