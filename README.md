Chapter 04 — Docker와 Fly.io 배포 학습 정리

Spring Boot URL 단축 서비스를 배포하는 과정을 중심으로, 헷갈리기 쉬운 개념과 확인할 실습을 정리한다.

이 저장물은 학습 노트와 실습 예시다. 아래 체크리스트는 직접 실행한 뒤 표시한다. Spring Boot 예시는 기존 Java 21·Gradle 프로젝트에 병합하는 용도이며, 전체 URL 단축 서비스 소스는 포함하지 않는다.

1. Dockerfile, 이미지, 컨테이너의 차이

구분

역할

예시

Dockerfile

이미지 생성 절차를 적는 파일

FROM, COPY, RUN

이미지

실행에 필요한 파일과 환경을 담은 템플릿

nginx-study:1.0

컨테이너

이미지로 생성하는 실행 단위

nginx-study-1

하나의 이미지로 여러 컨테이너를 만들 수 있다. 컨테이너는 이미지 전체를 단순 복사하는 것이 아니라 이미지 레이어를 공유하고 별도의 쓰기 가능한 레이어를 갖는다. 컨테이너를 삭제해도 이미지는 남는다.

Java 프로그램은 JAR만으로 실행되는 것이 아니라 JVM이 필요하다. 실행 이미지에 Java 런타임을 포함하면 서버마다 직접 Java를 설치하는 부담이 줄어든다. 다만 CPU 아키텍처, 환경변수, 네트워크, 외부 DB 등 실행 조건까지 자동 해결되는 것은 아니다.

2. 포트 매핑에서 헷갈린 부분

docker run -d --name nginx-study-1 -p 127.0.0.1:8080:80 nginx-study:1.0
docker run -d --name nginx-study-2 -p 127.0.0.1:8081:80 nginx-study:1.0

호스트 IP:호스트 포트:컨테이너 포트 순서다. 첫 번째 요청은 localhost:8080 → 컨테이너 내부 80으로 전달된다.

두 컨테이너 모두 내부 80번 포트를 사용할 수 있다.

같은 호스트 IP·프로토콜의 같은 포트를 동시에 할당하려 하면 충돌한다.

EXPOSE 80은 이미지의 포트 정보를 표시하며, 호스트 포트 공개는 -p로 지정한다.

127.0.0.1을 지정한 위 예시는 내 컴퓨터에서만 접근한다.

3. Nginx 실습

Docker Desktop 실행 후 이 README가 있는 폴더에서 시작한다.

cd nginx
docker build -t nginx-study:1.0 .
docker run -d --name nginx-study-1 -p 127.0.0.1:8080:80 nginx-study:1.0
docker run -d --name nginx-study-2 -p 127.0.0.1:8081:80 nginx-study:1.0
docker ps

브라우저에서 http://localhost:8080과 http://localhost:8081에 접속한다. 예상 결과: 같은 HTML 화면이 두 포트에서 표시된다.

index.html을 수정해도 이미 생성한 이미지에는 반영되지 않는다. 다시 빌드하고 기존 컨테이너를 새 이미지로 교체한다.

docker build -t nginx-study:1.1 .
docker rm -f nginx-study-1 nginx-study-2
docker run -d --name nginx-study-1 -p 127.0.0.1:8080:80 nginx-study:1.1
docker logs nginx-study-1

실습 종료 후 이번 실습 자원만 삭제한다.

docker rm -f nginx-study-1
docker rmi nginx-study:1.0 nginx-study:1.1
cd ..

4. Spring Boot 이미지에서 빌드와 실행을 나누는 이유

spring-examples/Dockerfile은 멀티 스테이지 빌드 예시다.

JDK가 있는 빌드 단계에서 기존 Gradle Wrapper로 clean build를 실행한다.

실행 단계에는 Java 런타임과 완성된 app.jar를 복사한다.

컨테이너가 시작되면 java -jar app.jar를 실행한다.

빌드 도구와 소스를 실행 이미지에 모두 남길 필요가 없다. 또한 프로젝트에 있는 Wrapper를 사용하면 프로젝트에서 정한 Gradle 버전을 따른다. gradlew, gradle/wrapper/gradle-wrapper.jar, gradle-wrapper.properties가 Git에 포함되어 있어야 한다.

build/libs/*.jar는 실행 JAR와 plain.jar를 동시에 선택할 수 있다. 예시에서는 bootJar의 파일명을 app.jar로 고정하고 그 파일 하나만 복사한다.

JVM 시스템 프로퍼티를 사용하는 다른 표현은 다음과 같다. -D 옵션은 -jar보다 앞에 둔다.

java -Dspring.profiles.active=prod -jar app.jar

5. 개발·운영 프로필

파일

적용 조건

예시 설정

application.yml

공통 설정

포트 8090, 기본 프로필 dev

application-dev.yml

dev 프로필

개발용 사이트 이름

application-prod.yml

prod 프로필

운영용 사이트 이름

같은 속성이 있으면 해당 프로필의 설정이 공통 설정을 덮어쓴다. 환경변수나 명령행 옵션처럼 파일보다 우선순위가 높은 설정도 있으므로 단순히 파일을 읽는 순서만으로 판단하지 않는다.

예시에서는 활성 프로필을 지정하지 않을 때 dev를 사용하고, 컨테이너에서는 SPRING_PROFILES_ACTIVE=prod를 지정한다. application-prod.yml은 운영 프로필에서 적용된다.

기존 프로젝트에 예시 적용하기

Dockerfile과 .dockerignore를 프로젝트 루트로 복사한다.

build.gradle.append.txt 내용을 기존 build.gradle에 병합한다.

세 YAML 파일의 설정을 src/main/resources/의 기존 설정에 병합한다. DB 등 기존 설정을 통째로 지우지 않는다.

컨트롤러의 package를 본인 프로젝트에 맞추고 메인 애플리케이션의 하위 패키지에 둔다.

기존 /study 매핑이 있다면 충돌하지 않게 바꾼다.

기존 spring.profiles.include: secret이나 필수 비밀 설정이 있다면 실행 환경에 맞게 검토한다. 실제 비밀값을 이미지에 복사하지 않는다.

Windows PowerShell에서 프로젝트 루트 기준:

.\gradlew.bat clean build
.\gradlew.bat bootRun --args='--spring.profiles.active=dev'

http://localhost:8090/study의 예상 결과: URL 단축 서비스 - 개발. Ctrl+C로 종료한 다음 컨테이너로 운영 설정을 확인한다.

docker build -t surl-study:1.0 .
docker run -d --name surl-study -p 127.0.0.1:8090:8090 surl-study:1.0
docker logs surl-study

같은 주소의 예상 결과: URL 단축 서비스 - 운영. 기존 프로젝트의 추가 의존성과 설정에 따라 별도 환경변수가 필요할 수 있다.

6. Fly.io와 GitHub Actions의 역할

Fly.io는 컨테이너 이미지를 바탕으로 앱을 실행하는 배포 환경이다.

fly.toml은 앱 이름, 내부 포트 등 배포 설정을 담는다.

GitHub Actions는 push 같은 이벤트에 맞춰 작업을 실행한다.

CI는 변경 사항의 빌드·테스트 등 통합 검증이며, CD는 배포 과정을 자동화하는 부분이다.

이 예시에서는 main push → Actions → flyctl deploy → 이미지 빌드 및 Gradle 검증 → 배포 순서로 진행한다. 별도 CI 작업은 없고 Docker 빌드 단계에서 Gradle의 기존 검증 작업을 실행한다.

적용 순서

Fly CLI 설치·로그인 후 프로젝트 루트에서 fly launch --no-deploy를 실행한다.

생성된 fly.toml에 예시 설정을 병합한다. 앱 이름은 실제 생성한 이름을 유지한다.

Spring Boot의 server.port와 Fly의 internal_port를 모두 8090으로 맞춘다.

fly deploy로 초기 배포 후 /study 응답과 로그를 확인한다.

앱용 deploy token을 발급하고 GitHub 저장소의 Actions Secret에 FLY_API_TOKEN으로 등록한다.

deploy.yml.example을 프로젝트 루트의 .github/workflows/deploy.yml로 저장한다.

main에 push하고 Actions 성공 여부와 실제 응답을 함께 확인한다.

fly auth login
fly launch --no-deploy
fly deploy
fly status
fly logs
fly tokens create deploy

토큰 출력은 캡처하거나 README에 넣지 않는다. 예시 워크플로에는 paths 필터가 없어 README 변경도 main에 push하면 배포가 실행된다. 브랜치명이 다르면 워크플로도 수정한다.

Fly.io 배포는 계정과 유료 자원 설정을 확인한 뒤 진행한다. 자동 정지 설정을 비용 0원 보장으로 이해하지 않는다. 예시의 최소 실행 머신 0 설정은 상시 가용성을 보장하는 구성이 아니다.

7. 비밀값을 Git에서 제외하는 것과 배포 시 전달하는 것은 별개

.gitignore는 새 파일 추적을 막는다. 이미 커밋한 비밀값이 과거 기록에서 사라지는 것은 아니다. 노출된 값은 교체해야 한다.

GitHub Secrets는 워크플로에 값을 전달하고, Fly Secrets는 앱 실행 시 환경변수로 값을 전달한다. GitHub Secrets에 담았더라도 파일로 복원해서 JAR에 넣으면 이미지 안에 포함될 수 있다.

이번 예시는 프로필 확인에 비밀값이 필요하지 않으므로 APPLICATION_SECRET_YML을 만들지 않는다. 실제 DB 비밀번호 등이 필요한 앱은 런타임 환경변수로 전달하도록 구성한다. 비밀값을 HTTP 응답으로 돌려주는 확인용 API는 만들지 않는다.

8. URL 정보를 메모리에만 저장할 때의 문제

URL 목록을 애플리케이션의 List나 Map에만 저장하면 프로세스 종료 후 다시 시작할 때 정보가 사라진다. 재배포도 이 상황을 만들 수 있다.

머신이 두 개이면 각 JVM의 메모리는 별개다. 머신 A에서 등록한 URL을 머신 B가 조회할 때 찾지 못할 수 있다. 재시작 문제뿐 아니라 데이터 공유 문제도 있다. 이를 해결하려면 앱 외부의 공통 DB 등에 저장해야 한다.

롤링 배포는 머신을 순차적으로 교체하는 전략이다. 머신 수, 헬스체크, 앱 시작 시간, 데이터 저장 방식에 따라 실제 가용성은 달라지므로 자동 배포를 설정했다고 곧바로 무중단을 보장한다고 쓰지 않는다.

9. 직접 확인할 항목

Nginx 이미지 빌드 성공

같은 이미지로 8080·8081에 컨테이너 두 개 실행

HTML 변경 후 이미지 재빌드·컨테이너 교체 확인

Spring Boot 로컬 빌드 성공

dev/prod에서 /study 응답 차이 확인

Fly 배포 후 /study 응답 확인

main push로 Actions 배포 성공 확인

제출할 캡처에 토큰·비밀값이 없는지 확인

실행 결과 기록: 직접 확인 후 아래 빈칸을 채운다.

확인 항목

실제 결과 / 캡처

두 컨테이너의 이름·포트



dev 응답 / prod 응답



Actions 실행 결과



Fly 앱 주소와 응답



참고 자료

수업: 멋쟁이사자처럼 PBL, Chapter 04 — fly.io로 서비스 배포하기

강의 저장소

Docker 포트 공개

Spring Boot Profiles

Fly.io GitHub Actions

Fly.io Secrets

Fly.io 배포 전략

강의 노션 텍스트와 공식 문서를 바탕으로 작성했다. 강의 GitHub 원본 파일은 이 자료 작성 환경에서 조회하지 못했으므로 패키지명과 기존 설정은 본인 프로젝트에서 확인한다.
