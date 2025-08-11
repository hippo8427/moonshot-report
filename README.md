# 중급 프로젝트 개인 개발 리포트 

**작성자**: 2팀 하상준  

**프로젝트명**: Moonshot  

**저장소** : [nb02-moonshot-team2](https://github.com/nb02-moonshot-team2/nb02-moonshot-backend)




<br>

## 프로젝트 개요

<br>

Moonshot은 사용자가 참여하는 프로젝트의 일정과 할 일을 체계적으로 관리할 수 있는 웹 서비스입니다.

프로젝트 생성, 멤버 초대, 할 일 등록·수정·삭제, 댓글 작성 등 협업에 필요한 기능을 제공하며, 칸반 보드와 캘린더 뷰를 통해 진행 상황을 직관적으로 파악할 수 있습니다.

또한, 정렬·필터·검색 기능을 통해 원하는 작업을 빠르게 찾을 수 있으며,

프로젝트 삭제나 유저 탈퇴 시 관련 데이터를 안전하게 처리하는 데이터 무결성 기능을 갖추고 있습니다.



<br><br>

## 담당 작업

<br>

**할 일 및 태그 기능 구현**  
  프로젝트 내 할 일을 등록, 수정, 삭제할 수 있도록 구현하였으며, 할 일별로 다중 태그를 지정할 수 있도록 설계하였습니다.

  할 일 목록은 담당자별, 상태별, 이름순, 기한임박순의 정렬 및 필터 기능을 지원하도록 구현했습니다.

**파일 업로드 기능 구현**

할 일에 첨부파일을 업로드할 수 있도록 기능을 구현하였으며, 업로드된 파일은 파이어베이스 스토리지에 보관되도록 처리했습니다.



<br><br>

## 기술적 성과

### 사용한 기술 스택

| 구분          | 기술명                                  |
| ----------- | ------------------------------------ |
| 런타임/프레임워크   | **Node.js**, **Express**             |
| 데이터         | **PostgreSQL**, **Prisma ORM**       |
| 인증/보안       | **bcrypt**, **JWT(또는 쿠키)**, **cors** |
| 파일          | **multer**, **firebase**             |
| 개발 환경/코드 품질 | **ESLint**, **Prettier**, **Husky**, **lint-staged** |


<br>

### 주요 구현 기능

1. 할 일 등록, 수정, 삭제, 상세 조회 기능

2. 다중 태그 지정 기능

3. 필터/정렬/검색/페이지네이션 기능
 
4. 파일 첨부 기능(확장자·용량 검증 포함)

<br><br>

## 트러블 슈팅

### 첨부파일 단일 업로드 제한

#### 구현 방식
1.  multer의 upload.single('file') 메서드를 사용하여 단일 파일만 업로드 가능.

2. 업로드된 파일은 req.file 객체로 받아 Firebase Storage에 저장하고, 서명 URL(Signed URL)을 생성하여 응답으로 전달.

3. 파일명은 원본 파일명 + 업로드 시각(timestamp) + UUID 조합으로 생성해 중복 방지.

#### 문제점
1. 프론트엔드에서는 여러 파일을 동시에 업로드할 수 있도록 UI가 구현되어 있었으나, 백엔드에서 단일 파일만 허용.
  
2. 요청 필드명(file)이 프론트 요구사항(files)과 불일치.

#### 해결 방법
1. Multer 설정을 upload.array('files')로 변경해 다중 업로드 허용.

2. 컨트롤러에서 req.files를 배열로 받아 Promise.all을 사용해 병렬 업로드 처리.

3. 파일명 생성, Firebase 업로드, URL 생성 과정을 배열 순회하며 처리.

4. 응답 형식을 { data: uploadedUrls } 형태로 변경해 여러 파일 URL 반환.

```js
router.post('/', upload.array('files'), uploadFiles);

const files = req.files as Express.Multer.File[];
if (!files) throw { status: statusCode.badRequest, message: errorMsg.wrongRequestFormat };

const uploadedUrls = await Promise.all(
  files.map(async (file) => {
    const { originalname, buffer, mimetype } = file;
    const ext = path.extname(originalname);
    const baseName = path.basename(originalname, ext);
    const safeBaseName = baseName.replace(/[^a-zA-Z0-9_-]/g, '_');
    const timestamp = Date.now();
    const firebaseFileName = `${safeBaseName}_${timestamp}_${randomUUID()}${ext}`;
    const destination = `files/${firebaseFileName}`;

    const fileRef = bucket.file(destination);
    await fileRef.save(buffer, { metadata: { contentType: mimetype } });

    const [url] = await fileRef.getSignedUrl({ action: 'read', expires: '03-01-2500' });
    return url;
  })
);

return res.status(200).json({ data: uploadedUrls });
```

<br>

### orderBy 네이밍 컨벤션 오류

#### 구현 방식
1. 프론트엔드에서 할 일 목록을 정렬할 때, orderBy 쿼리 파라미터를 전송.

2. 백엔드에서 해당 값을 taskRepository.getAllTasks()의 orderBy 옵션으로 전달.

3. Prisma ORM이 DB 컬럼명을 기반으로 정렬을 수행 (createdAt, dueDate 등).

#### 문제점
1. 프론트엔드에서 endDate라는 키를 보내지만, DB 컬럼명은 dueDate.

2. Prisma가 인식하지 못하는 필드를 전달하면 쿼리 에러 발생.

3. 컬럼명이 불일치해 정렬 기능이 동작하지 않음.

#### 해결 방법

1. 백엔드에서 프론트 orderBy 값을 내부 컬럼명으로 변환하는 매핑 함수 추가.

2. endDate가 들어오면 dueDate로 변경, 나머지는 그대로 전달.

```js
// 매핑 함수 추가
const mapOrderByKey = (key: string) => (key === 'endDate' ? 'dueDate' : key);

const result = await taskRepository.getAllTasks({
  projectId,
  userId,
  status,
  assignee,
  keyword,
  order,
  orderBy: mapOrderByKey(orderBy) as TaskOrderBy, // 매핑 적용
  skip,
  take: limit,
});
```
- 스키마를 변경하는 방법도 있지만 매핑으로 해결해보고 싶었습니다.

```js
// status 를 isDone 으로 변환
    if ('status' in data) {
      data.isDone = data.status === 'done';
      delete data.status;
    }
```
- 같은 방법으로 하위 할일의 체크박스 기능 오류를 수정했습니다.

<br>

### 할 일 상세 페이지에서 이미지 첨부 불가

#### 구현 방식
1. 할 일 업데이트 API는 날짜·상태·담당자·태그·첨부파일까지 한 요청에서 함께 수정하도록 설계.

2. 날짜는 startYear/Month/Day, endYear/Month/Day 를 받아 startedAt, dueDate 로 변환하여 저장.

3. 첨부파일은 프론트에서 넘어온 URL 배열을 받아 DB(taskFiles)에 매핑 후 응답으로 반환.

#### 문제점
 1. 첨부파일만 업데이트하는 경우 날짜 미전달로 에러
    - 프론트는 첨부파일만 수정할 때 날짜 필드를 보내지 않음 > new Date(undefined, …)가 되어 변환 에러.
      
 3. 응답 스키마 불일치(배열 형태 차이)
    - 프론트는 attachments: string[](URL 배열)을 기대했지만,
    - 백엔드는 { id, url }[] 형태로 반환
   
#### 해결 방법
1. 날짜 옵셔널 처리 (값이 있을때만 업데이트)

```js
const startedAt =
  startYear && startMonth && startDay
    ? new Date(startYear, startMonth - 1, startDay)
    : undefined;

const dueDate =
  endYear && endMonth && endDay
    ? new Date(endYear, endMonth - 1, endDay)
    : undefined;
```

2. 응답 방식 통일

```js
return {
attachments: task.taskFiles.map((file) => file.fileUrl), // ← string[]
}
```

#### 개선점
옵셔널은 현재의 오류만 해결해줄 뿐, 좋은 대안은 아닌 것 같다는 생각이 들어 작성합니다.
1. 프론트에서 항상 날짜를 보내도록 강제하는 방법
   
2. 값이 없다면, db에서 기존 날짜값을 읽어와 유지하는 방법

<br>

### 이미지 삭제 버튼 표시 안 됨

#### 문제점
1. flex-grow: 1 만 적용된 상태에서 긴 파일명일 경우, 파일명 영역이 버튼 영역을 침범해 버튼이 사라진 것처럼 보이는 현상 발생.

#### 해결 방법

1. .fileName의 flex-grow: 1을 **flex: 1;**로 변경.

2. flex: 1;은 flex-grow: 1; flex-shrink: 1; flex-basis: 0; 세 속성을 한 번에 지정하는 축약형.

3. 이를 통해 .fileName은 공간을 균등하게 차지하면서도, 삭제 버튼이 항상 표시될 수 있도록 공간이 보장됨.

```js
.fileName {
  flex: 1; // flex-grow:1; flex-shrink:1; flex-basis:0; 
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
}
```

<br>

### 상태 수정 시 태그·첨부파일이 삭제되던 문제

#### 구현 방식
1. 기존 업데이트 로직에서 taskTags와 taskFiles는 항상 deleteMany → create 패턴으로 처리.

2. 요청에 태그나 첨부파일이 없으면 deleteMany로 모두 삭제된 뒤 새 값이 없으니 빈 상태로 저장됨.

3. 기본값을 []로 설정하여 attachments = [], tags = []가 요청 시 자동 들어오도록 함.

#### 문제점
1. 다른 필드 (상태, 내용) 만 수정해도 기본값 [] 때문에 기존 태그·첨부파일이 전부 삭제됨.

2. 사용자는 단순 상태 변경만 했는데, 기존 첨부파일과 태그가 전부 사라지는 손실 발생.

#### 해결 방법
1. tags나 attachments가 undefined일 때는 수정 로직을 스킵하게 변경.

2. 기본값 [] 제거 → 요청에서 값이 빠지면 undefined로 인식.

```js
// repository
taskTags: tags !== undefined
  ? {
      deleteMany: {},
      create: tags.map((tagName) => ({
        tag: {
          connectOrCreate: {
            where: { tag: tagName },
            create: { tag: tagName },
          },
        },
      })),
    }
  : undefined,

taskFiles: attachments !== undefined
  ? {
      deleteMany: {},
      create: attachments.map((file) => ({
        fileName: file.name,
        fileUrl: file.url,
      })),
    }
  : undefined,

```

### description 필드 추가

#### 문제점
1. API 명세서에는 description 필드가 없었으나, 프론트엔드 예시에서는 할일 상세 페이지에 내용을 표시하고 있었음.

### 해결 방법

1. description 필드 추가
 
2. 각 기능 응답 객체에 description 필드를 포함시켜 프론트 표시 요구사항 충족.

<br><br>

## 협업 및 피드백

1. 개발 규칙을 팀 위키에 정리해 두어, 모든 팀원이 참고하기 편했고 규칙의 일관성을 유지할 수 있었습니다.

2. ESLint, Prettier, Husky, lint-staged 사용을 제안하고 적용했습니다.
   
   초기에는 설정이 과도하여 개발 속도에 영향을 주었으나, 일부 규칙을 완화하여 실용성을 높였습니다.

3. PR 작성 시 리뷰어를 지정하고, 리뷰를 받은 뒤 본인이 직접 머지하는 방식으로 진행했습니다.
   
   리뷰 과정에서 적극적으로 소통하며 문제점을 빠르게 보완할 수 있었습니다.

4. 테스트를 진행한 후 수정이 필요한 부분을 프로젝트 계획서 하단에 팀원들이 기록해주어서, 트러블슈팅 과정이 훨씬 수월했습니다.

<br><br>

## 코드 품질 및 최적화


<br><br>

## 향후 개선 사항

1. Prisma 자동 생성 타입 활용

   현재는 직접 정의한 타입을 사용했으나, Prisma가 제공하는 자동 생성 타입을 활용하면 타입 안정성과 유지보수성이 향상될 것으로 예상됩니다.

2. 초기 설계 및 명세 파악 강화

   코드 작성 후 테스트·수정 과정에서 코드가 점점 복잡해졌습니다.

   프로젝트 초기에 API 명세서와 프론트엔드 예시를 충분히 분석하여 설계 단계에서 구조를 깔끔히 잡는 것이 중요하다고 느꼈습니다.

3. 클래스 기반 구조 적용

   현재는 req, res, next를 활용한 객체 리터럴 방식만 사용했습니다.

   향후에는 클래스 기반 서비스·컨트롤러 구조를 적용해 확장성과 재사용성을 높이고 싶습니다.

4. 중복 인증 로직의 미들웨어화

   인증 관련 검증 로직이 여러 곳에 중복되어 있습니다.

   이를 공통 미들웨어로 분리하면 코드 중복을 줄이고 가독성을 개선할 수 있습니다.

5. 페이지네이션 심화 학습

   페이지네이션 로직에 대한 이해가 부족하다고 느꼈습니다.

   다양한 페이지네이션 방식(오프셋·커서 기반 등)과 성능 최적화 기법에 대해 추가 학습이 필요합니다.

6. 트랜잭션 관리 적용

   데이터 무결성과 일관성을 보장하기 위해, 여러 쿼리를 묶어 처리하는 트랜잭션 로직을 적극적으로 적용할 계획입니다.

   예를 들어, 프로젝트 생성과 멤버 추가를 하나의 트랜잭션으로 묶어 처리하면 안정성이 향상됩니다.




