# 중급 프로젝트 개인 개발 리포트 

**작성자**: 2팀 하상준  

**프로젝트명**: Moonshot  

**저장소** : [nb02-moonshot-team2](https://github.com/nb02-moonshot-team2/nb02-moonshot-backend)




<br>

## 1. 프로젝트 개요

<br>

Moonshot은 사용자가 참여하는 프로젝트의 일정과 할 일을 체계적으로 관리할 수 있는 웹 서비스입니다.

프로젝트 생성, 멤버 초대, 할 일 등록·수정·삭제, 댓글 작성 등 협업에 필요한 기능을 제공하며, 칸반 보드와 캘린더 뷰를 통해 진행 상황을 직관적으로 파악할 수 있습니다.

또한, 정렬·필터·검색 기능을 통해 원하는 작업을 빠르게 찾을 수 있으며,

프로젝트 삭제나 유저 탈퇴 시 관련 데이터를 안전하게 처리하는 데이터 무결성 기능을 갖추고 있습니다.



<br><br>

## 2. 담당 작업

<br>

**할 일 및 태그 기능 구현**  
  프로젝트 내 할 일을 등록, 수정, 삭제할 수 있도록 구현하였으며, 할 일별로 다중 태그를 지정할 수 있도록 설계하였습니다.

  할 일 목록은 담당자별, 상태별, 이름순, 기한임박순의 정렬 및 필터 기능을 지원하도록 구현했습니다.

**파일 업로드 기능 구현**

할 일에 첨부파일을 업로드할 수 있도록 기능을 구현하였으며, 업로드된 파일은 파이어베이스 스토리지에 보관되도록 처리했습니다.



<br><br>

## 3. 기술적 성과 및 해결

### 사용한 기술 스택

| 구분        | 기술명                                              |
| --------- | ---------------------------------------------------|
| 런타임/프레임워크 | **Node.js**, **Express**                             |
| 데이터       | **PostgreSQL**, **Prisma ORM**           |
| 인증/보안     | **bcrypt**, **JWT(또는 세션+쿠키)**, **cors**  |
| 파일        | **multer**, **firebase** |


<br>

### 1. 답글 등록, 수정, 삭제 기능 구현

<br>

[ [ **Comment Wiki** ] ](https://github.com/gyunam-bark/nb02-how-do-i-look-team1/wiki/%5BAPI%5DComment) 

**comment-controller.js** : 클라이언트 요청을 받아 서비스 코드 호출 및 응답을 처리

![image](https://github.com/user-attachments/assets/4352edff-5f67-412b-a77d-815a6a7d27d0)

 <br>
 
**comment-service.js** : 답글 등록, 수정, 삭제 기능의 핵심 비즈니스 코드와 DB 접근 로직 구현

![image](https://github.com/user-attachments/assets/ca306fd0-b327-45f3-b2b9-1f723aa440ab)
  
<br>

**comment-route.js**: 각 기능별 API 엔드포인트 구성 및 미들웨어 연동을 통한 유효성검사 적용

![image](https://github.com/user-attachments/assets/af113533-776b-43c3-939c-6d51cc1743dc)

<br><br>

### 2. 비밀번호 해싱 기능

<br>

[ [ **Password Hashing Wiki** ] ](https://github.com/gyunam-bark/nb02-how-do-i-look-team1/wiki/%5B%EB%AF%B8%EB%93%A4%EC%9B%A8%EC%96%B4,-%EC%9C%A0%ED%8B%B8%5D-%EB%B9%84%EB%B0%80%EB%B2%88%ED%98%B8-%ED%95%B4%EC%8B%B1)

**hash-password.js** : 비밀번호를 bcrypt로 해싱 처리 
**compare-password.js** : 입력 비밀번호와 저장된 해시 비교

![image](https://github.com/user-attachments/assets/273d63e0-d93f-4764-9332-7b1d5d910e5f)
- 해싱 기능과 비교 기능을 나눴습니다.

<br>

**bcrypt-middleware.js**: 해싱 기능을 미들웨어로 분리하여 재사용성과 유지보수 용이성 확보
 
![image](https://github.com/user-attachments/assets/2e32d091-0440-410a-a126-a42d7d33eab7)
- 입력된 비밀번호가 존재할 경우, 요청 본문(req.validated.body)의 password를 해싱 하도록 했습니다.



<br><br>

## 4. 문제점 및 해결 과정

### 스타일 비밀번호 참조 경로 수정

<br>

초기엔 **Comment** 에서 **StyleId** 를 참조하려 했으나, 이는 다음과 같은 문제를 야기했습니다

1. 답글이 스타일에도 의존하게 되어서 불필요한 의존성이 늘어났습니다.
2. 스타일과 큐레이션을 동시에 참조하면, 의도치않은 참조 충돌이 일어날 수 있습니다.

따라서 답글은 오직 큐레이션을 통해서만 스타일 정보를 간접적으로 접근하도록 설계했습니다.  

<br>

![image](https://github.com/user-attachments/assets/769b034b-d57f-4fe1-b9fd-70e6f586e9b9)
- 기능간의 책임을 명확히 구분하고, 데이터 결합도를 낮추기 위해 style 비밀번호가 필요하더라도  
  comment 는 curation 을 통해서만 style 정보에 접근 할 수 있도록 했습니다.

  <br>
  
![image](https://github.com/user-attachments/assets/500f405c-e0cb-48b2-adef-8f3edfddd37a)
- 반환값에 쓰일 닉네임을 가져올 때에도 curation 을 거칩니다.

### 수정 결과  

결합도와 영향범위를 고려하여 **Style** 정보를 **Curation** 을 통해 간접 접근하도록 변경함으로써, 기능 간 결합도를 낮추고, 구조변경이나 삭제같은 작업이 댓글 기능에 직접적인 영향을 미치지 않게 개선하였습니다.

또한, 직접 참조하지 않아도  **Curation** 을 이용한 간접 참조를 통해 어떤 스타일 글에 대한 댓글인지 명확히 추적이 가능해졌습니다.


<br><br>

## 5. 협업 및 피드백

<br>

팀원들과 API 명세서 기반의 역할 분담을 통해 충돌 없이 개발을 진행하였습니다.

코드 리뷰를 통해 유틸 함수 분리나 코드 개선 등 여러 피드백을 반영하였습니다.

특히, 피드백을 통해 단일 파일에 집중되었던 해싱 로직을 기능별로 분리하고, 라우터 단위 미들웨어로 적용하는 방식으로 개선한 부분은 협업 과정에서 얻은 가장 실질적인 성과 중 하나였습니다.

이 과정에서 **반복되는 로직을 함수로 분리하는 이유**, 그리고 **의미 있는 변수명을 명확히 정하는 중요성**을 체감했습니다.  

예를 들어, `const saltRounds = 10;`이라는 단순해보이는 상수 선언도, **의도를 담은 이름을 통해 코드의 의미를 명확히 전달하고 유지보수에 유리** 하다는 것을 알게 되었습니다.

또한, 단순한 `bcrypt.compare()` 호출도 반복해서 사용되기 때문에, 이를 별도 유틸 함수로 분리하면 위의 변수명 선언과 더불어  **중복 제거뿐 아니라 코드의 재사용성까지 함께 확보**할 수 있다는 점을 실감했습니다.

이러한 피드백과 개선 과정을 통해, **코드는 단순히 동작만 하면 되는 것이 아니라, 함께 일하는 개발자들이 함께 이해하고 유지할 수 있도록 구조화해야 한다**는 것을 느낄 수 있었습니다.



<br><br>

## 6. 코드 품질 및 최적화

### bcrypt 구조 리팩토링

<br>

[ [ **리팩토링 PR** ] ](https://github.com/gyunam-bark/nb02-how-do-i-look-team1/pull/185)

초기엔 모든 해싱 로직을 한 파일에 구현하여 기능 분리가 불분명하고, 유지보수 및 재사용성이 떨어졌습니다. 

<br>

***초기코드***

![image](https://github.com/user-attachments/assets/12ca898e-7a57-4077-8d33-17d90a564795)

팀장님의 제안으로 이를 해결하기 위해 다음과 같은 개선을 진행했습니다:


- **기능별로 파일 분리** : 해싱 기능을 각각 **hash-password.js**와 **compare-password.js**로 분리

- **기능 단위의 미들웨어 적용** : 해싱이 필요한 API 요청에만 선택적으로 적용할 수 있도록, 해싱 기능을 미들웨어로 분리

<br>

#### 디렉토리 구조 예시
```
project/
├── middlewares/
│   └── bcrypt-middleware.js
├── utils/
│   ├── hash-password.js
│   └── compare-password.js

```

<br><br>

## 7. 향후 개선 사항 및 제안

<br>

**비밀번호 인증 개선**
  - 현재는 평문 비밀번호 비교 기반이므로, 향후 JWT 등을 이용한 인증 시스템으로의 전환이 필요합니다.

**강제 검증 툴 사용**  
  - Husky와 lint-staged 같은 Git Hook 기반 검증 툴을 도입하여 커밋 단계에서 검사를 강제 실행함으로써, 코드의 일관성과 품질을 더욱 견고하게 확보할 필요가 있습니다.



