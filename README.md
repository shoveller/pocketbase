- 포켓베이스는 어디에나 설치할수 있는 바이너리를 제공하지만, 도커 파일이나 도커 이미지는 제공하지 않는다.
- 도커파일을 깃허브에 올리면 CI/CD 는 물론, 프라이빗 도커 허브를 가지게 된다.

# 레파지토리 생성
- web ui 레파지토리를 생성하고 템플릿 리파지토리로 설정한다.
	- 템플릿 리파지토리의 소스코드는 원본과 완전히 분리된다는 점에서 fork 와 다르다.
![스크린샷 2025-05-01 오후 5 58 05](https://github.com/user-attachments/assets/81aa4045-665f-44a8-8511-9fcfd17e6006)


# 도커파일 추가
- 프로젝트 루트에 Dockerfile을 추가한다.
	- 환경변수를 추가해서 빌드 워크플로우로 제어할 수 있게 한다.
```Dockerfile
FROM alpine:latest
# 환경변수를 사용하면 외부에서 주입을 받게 설계할 수 있다.
ARG PB_VERSION=0.27.2

# openssh 를 설치하면 파일을 백업할때 송수신이 편하다.
RUN apk add --no-cache unzip openssh

ADD https://github.com/pocketbase/pocketbase/releases/download/v${PB_VERSION}/pocketbase_${PB_VERSION}_linux_amd64.zip /tmp/pb.zip
RUN unzip /tmp/pb.zip -d /pb/

EXPOSE 8080
CMD ["/pb/pocketbase", "serve", "--http=0.0.0.0:8080"]
```

# 빌드 워크플로우 추가
- `.github/workflows/docker-publish.yml` 을 추가한다.
	- `workflow_dispatch` 옵션을 추가해서 수동으로 빌드할 수 있게 한다.
	- 동적으로 태그를 발행해서 이미지가 겹치지 않게 한다.
	- `secrets.GITHUB_TOKEN` 은 기본으로 제공되는 엑세스 토큰이다. 이것을 사용해서 깃허브 컨테이너 레지스트리(ghcr) 에 배포한다.
	- 유저 시크릿과 환경변수를 연결해서 시크릿 수정만으로 업그레이드를 할수 있게 한다.
```yml
name: Build and push Docker image

on:
  push:
    branches: [ main ]
  workflow_dispatch:  # 수동 실행 가능

jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Generate tag (YYYYMMDD-RUN_NUMBER)
        id: tag
        run: echo "value=$(date +%Y%m%d)-${{ github.run_number }}" >> $GITHUB_OUTPUT

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/pocketbase:latest
            ghcr.io/${{ github.repository_owner }}/pocketbase:${{ steps.tag.outputs.value }}
          build-args: |
            PB_VERSION=${{ secrets.PB_VERSION }}
```

![스크린샷 2025-05-01 오후 6 03 36](https://github.com/user-attachments/assets/3c5b75eb-e052-476d-86bb-a29821c4e367)


# 확인
- 빌드에 성공하면 `packages` 메뉴가 활성화된다.
![스크린샷 2025-05-01 오후 6 14 23](https://github.com/user-attachments/assets/bc346719-6945-4eb4-a7de-dbbc59073d56)



- 사용법이 이곳에 나와있으니 참고하자
![스크린샷 2025-05-01 오후 6 14 55](https://github.com/user-attachments/assets/6a22347d-90fd-445c-8fdd-444194a3f60f)
