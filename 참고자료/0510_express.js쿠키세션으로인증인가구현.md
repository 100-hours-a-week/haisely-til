# [05/10] express.js 쿠키/세션으로 인증 인가 구현

목표 : 커뮤니티 개발 중 쿠키/세션을 통해 인증 인가를 구현하기

# 기본 설정

app.js에 서버에서 세션을 사용함을 명시한다.

```jsx
app.use(
    session({
        secret: 'yourSecretKey',
        saveUninitialized: true,
        resave: false,
        cookie: {
            maxAge: 24 * 60 * 60 * 1000, // 쿠키 유효 시간 (예: 1일)
        },
    }),
);
```

인증하는 함수를 일괄적으로 쓰려면 아래처럼 사용하면 된다.

```jsx
// 일괄 적용 코드
// 미들웨어: 모든 요청에 대해 실행됨
app.use((req, res, next) => {
    // 여기서 인증 및 인가 로직을 구현
    // 예를 들어, 쿠키 또는 세션을 확인하여 사용자를 인증하고 해당 권한을 확인할 수 있음
    // 인증 및 인가가 실패하면 적절한 응답을 보내거나 요청을 중단할 수 있음
    // 만약 성공하면 다음 미들웨어로 넘어갈 수 있음
    next();
})
```

# 인증 미들웨어 함수 만들기

하지만, 로그인이나 로그아웃 api라든가… 아니면 인증 없이도 읽을 수 있는 페이지가 있을 수 있으므로 미들웨어 함수를 api마다 적용하기로 결정했다. 인증을 위해, session과 session에 저장된 user정보를 확인하고, 인증이 안 되면 /login으로 리다이렉트 한다.

```jsx
const authenticateMiddleware = (req, res, next) => {
    if (!req.session || !req.session.user) {
        res.redirect('/login').status(401).json(makeRes(401, "unauthorized", null));
        return;
    }
    next();
};

module.exports = {
    authenticateMiddleware
}
```

그러면 이제 login을 할 때 session에 user정보를 저장해야한다. 로그인이 성공적으로 된 경우, `req.session.user = user;`를 적어둔다.

```jsx
const login = (req, res) => {
    const requestData = req.body;
    if(!requestData.email){res.status(400).json(makeRes(400, "invalid_user_email", null)); return;} // invalid email
    if(!requestData.password){res.status(400).json(makeRes(400, "invalid_user_password", null)); return;} // invalid password
    let user = findUserByEmail(requestData.email);
    if(!user){res.status(401).json(makeRes(401, "user_not_found_for_email", null)); return;} // no user
    if(user.password !== requestData.password){res.status(401).json(makeRes(401, "incorrect_password", null)); return;} // incorrect password
    delete user.password;
    req.session.user = user;
    user.auth_token = req.sessionID;
    res.status(200).json(makeRes(200, "login_success", {"user" : user}));
}
```

“/boards/”를 라우팅하는 코드의 미들웨어로 방금 만들었던 `authenticateMiddleware` 를 넣어둔다.

```jsx
const {authenticateMiddleware} = require('../controllers/AuthenticationUtils');

// 게시글 목록 조회
router.get('/',  authenticateMiddleware, boardController.getBoards);
```

이제 쿠키를 다 지우고, 프론트엔드에서 로그인을 해본다.

## 문제 발생 1

- 결과 사진 (로그인이 된 후 /boards로 이동된 뒤의 문제)

![스크린샷 2024-05-10 오전 11.01.29.png](%5B05%2010%5D%20express%20js%20%E1%84%8F%E1%85%AE%E1%84%8F%E1%85%B5%20%E1%84%89%E1%85%A6%E1%84%89%E1%85%A7%E1%86%AB%E1%84%8B%E1%85%B3%E1%84%85%E1%85%A9%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%8C%E1%85%B3%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%80%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%92%206d78c38a66554bea80eb7d2a5d900043/ddd.png)

역시 호락호락하지 않다. 당연히 한 번에 될 거라고 생각 안 했으니 ㄱㅊ다. 프론트에서 보면 일단 서버쪽 응답에 문제가 있어보이니 서버쪽 로그를 봐본다.

```jsx
0|app      | TypeError: Cannot read properties of undefined (reading 'status')
// AuthenticationUtils.js에서 발생
```

왜 못읽지… 찾아보니 redirect 함수 때문인 것 같아서 이 부분을 빼줬다. 

## 해결 방법 1

```jsx
const authenticateMiddleware = (req, res, next) => {
    if (!req.session || !req.session.user) {
        return res.status(401).json(makeRes(401, "unauthorized", null));
    }
    next();
};

module.exports = {
    authenticateMiddleware
}
```

- 적용 후 사진 (다시 로그인하고 /boards)

![Untitled](%5B05%2010%5D%20express%20js%20%E1%84%8F%E1%85%AE%E1%84%8F%E1%85%B5%20%E1%84%89%E1%85%A6%E1%84%89%E1%85%A7%E1%86%AB%E1%84%8B%E1%85%B3%E1%84%85%E1%85%A9%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%8C%E1%85%B3%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%80%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%92%206d78c38a66554bea80eb7d2a5d900043/Untitled.png)

생각해보니 애초에 로그인을 했는데 if문에 들어갔다는게 문제였다 ㅇㅇ 그래도 일단 status는 잘 읽어서 보여준다. `console.log(req.session)`을 해보니 `undefined`가 출력됐다. 

![Untitled](%5B05%2010%5D%20express%20js%20%E1%84%8F%E1%85%AE%E1%84%8F%E1%85%B5%20%E1%84%89%E1%85%A6%E1%84%89%E1%85%A7%E1%86%AB%E1%84%8B%E1%85%B3%E1%84%85%E1%85%A9%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%8C%E1%85%B3%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%80%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%92%206d78c38a66554bea80eb7d2a5d900043/Untitled%201.png)

그렇다. 애초에 쿠키는 저장도 되지 않았다… 생각해보니 백엔드에 seassion 코드를 추가하는 것 외에 쿠키 설정이라든지… 이런 걸 전혀 하지 않았다. 찾아보니 해결 방법은 다음과 같다.

## 해결 방법 2

### 1. 백엔드 서버 cors 설정하기

```jsx
app.use(cors({
    origin: ['http://localhost:3000'], 
    credentials: true  
}));
```

### 2. 프론트엔드 서버 fetch 함수 설정하기

```jsx
const response = await fetch(address, {
            method: 'POST',
            credentials: 'include',  // <------- 추가됨
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(jsonData)
        });
        return response.json();
                }catch(error) {
        console.error('Error:', error);
    }
}
```

`credentials` 설정을 해주었는데 이걸 해줘야 쿠키까지 포함해서 응답을 받을 수 있다.

![Untitled](%5B05%2010%5D%20express%20js%20%E1%84%8F%E1%85%AE%E1%84%8F%E1%85%B5%20%E1%84%89%E1%85%A6%E1%84%89%E1%85%A7%E1%86%AB%E1%84%8B%E1%85%B3%E1%84%85%E1%85%A9%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%8C%E1%85%B3%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%80%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%92%206d78c38a66554bea80eb7d2a5d900043/Untitled%202.png)

쿠키도 잘 왔고 로그인을 했을 때 boards 페이지를 잘 fetch해냈다!

## 로그아웃 함수 만들기

```jsx
const logout = (req, res) => {
    req.session.destroy(error => {
        if (error) {
            return res.status(500).json(makeRes(500, "로그아웃 중 문제가 발생했습니다.", null));
        }

        return res.status(200).json(makeRes(200, null, null)).redirect('/login');
    });
}
```

![Untitled](%5B05%2010%5D%20express%20js%20%E1%84%8F%E1%85%AE%E1%84%8F%E1%85%B5%20%E1%84%89%E1%85%A6%E1%84%89%E1%85%A7%E1%86%AB%E1%84%8B%E1%85%B3%E1%84%85%E1%85%A9%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%8C%E1%85%B3%E1%86%BC%20%E1%84%8B%E1%85%B5%E1%86%AB%E1%84%80%E1%85%A1%20%E1%84%80%E1%85%AE%E1%84%92%206d78c38a66554bea80eb7d2a5d900043/Untitled%203.png)

로그아웃을 한 다음에, 직접 /boards url에 접속하려할 때 401 상태를 성공적으로 보낸다!

## 인증받지 못한 경우, 로그인 화면으로 서빙하기

지금은 인증을 못 받았으면 그냥 /boards 페이지에서 글을 못 불러오기만 한다.

따라서 인증 받지 못한 경우, /login 페이지로 바로 넘어갈 수 있도록 해볼 것이다.

생각보다 간단하게 구현할 수 있었다! fetch하는 함수에 일괄적으로 상태코드가 401인 경우를 처리하도록 코드를 넣었다.

```jsx
async function fetchData(path) {
    const address = getBackendDomain() + path;
    try {
        const response = await fetch(address,
            {
                method: 'GET',
                credentials: 'include'
            });
        **if (response.status === 401) {**        // <---- 추가됨
            **window.location.href = '/login';     
        }**                                     // <---- 여기까지
        return response.json(); // JSON 데이터 반환
    } catch (error) {
        console.error('Error fetching data:', erroㄴr);
        throw error; // 에러 처리
    }
}
```

이제 인증 미들웨어함수가 필요한 api에 넣어주었다. 아래는 user routing 코드 부분만 가져왔다.

```jsx
// 로그인
router.post('/login', userController.login);  // <-- 인증 필요 x

// 회원가입
router.post('/signup', userController.signUp);   // <-- 인증 필요 x

// 로그아웃
router.post('/logout', authenticateMiddleware, userController.logout);

// 회원 정보 조회
router.get('/:id', authenticateMiddleware, userController.getUserById);

// 회원 정보 수정
router.patch('/:id', authenticateMiddleware, userController.patchUser);

// 비밀번호 변경
router.patch('/:id/password', authenticateMiddleware, userController.patchPassword);

// 회원 정보 삭제
router.delete('/:id', authenticateMiddleware, userController.deleteUser);

// 로그인 상태 확인
router.get('/auth/check', authenticateMiddleware, userController.authCheck);

// 이메일 중복 체크
router.get('/email/check', userController.emailCheck);

// 닉네임 중복 체크
router.get('/nickname/check', userController.nicknameCheck);
```

## 세션의 user 데이터 사용하기

이전에 user 관련 데이터는 더미로 넣어뒀었는데 드디어 고칠 때가 됐다. before→after만 간단하게 넣어두겠다.

- makeBoards
    
    **before**
    
    ```
    // need to send user data
    const newPost = makeNewBoard(null, requestData.postTitle, requestData.postContent, requestData.attachFilePath);
    
    const makeNewBoard = (user, postTitle, postContent, attachFilePath) => {
        // need to use user data
        return {
            "post_title": postTitle,
            "post_content": postContent,
            "user_id": 1,  // 수정
            "nickname": "테스트",  // 수정
            "created_at": getTimeNow(),
            "updated_at": getTimeNow(),
            "comment_count": "0",
            "hits": "1",
            "file_path": attachFilePath  || null,
            "profile_image_path": "/images/default.png"  // 수정
        };
    ```
    
    **after**
    
    ```jsx
    const user = req.session.user
    const newPost = makeNewBoard(user, requestData.postTitle, requestData.postContent, requestData.attachFilePath);
    
    const makeNewBoard = (user, postTitle, postContent, attachFilePath) => {
        return {
            "post_title": postTitle,
            "post_content": postContent,
            "user_id": user.user_id,  
            "nickname": user.nickname, 
            "created_at": getTimeNow(),
            "updated_at": getTimeNow(),
            "comment_count": "0",
            "hits": "1",
            "file_path": attachFilePath  || null,
            "profile_image_path": user.profile_image ||"/images/default.png" 
        };
    }
    ```
    
- getUserById
    
    **before**
    
    ```jsx
    const getUserById = (req, res) => {
        const userId = req.params.id;
        console.log(userId);
        let user = findUserById(userId);
        if(!user) {res.status(404).json(makeRes(404, "not_found_user", null)); return;}  // user not found
        delete user.password;
        res.status(200).json(makeRes(200, null, {"user" : user}));
    }
    ```
    
    **after**
    
    ```jsx
    const getUserById = (req, res) => {
        const userId = req.session.user.user_id;
        console.log(userId);
        let user = findUserById(userId);
        if(!user) {res.status(404).json(makeRes(404, "not_found_user", null)); return;}  // user not found
        delete user.password;
        res.status(200).json(makeRes(200, null, {"user" : user}));
    }
    ```
    

# 인가 미들웨어 함수 만들기

~~만든 api 중, 1. 특정 데이터(user, board, comment)를 patch, delete하거나 2. user data를 get할 때 특정 사용자만 요청할 수 있도록 하는 인가가 필요하다!~~

위처럼 생각했는데 user 정보 수정할 때 userId를 클라이언트에서 값으로 받는게 아니라 `session.user.userId`로 가져오기 때문에 이 부분 인가를 따로 구현한다는 것 자체가 애매해졌다…

따라서 board와 comment 부분만 작성자를 확인해서 `session.user.userId`와 같은지를 봐야 할 듯 하다!

*추가로 인가를 인증과 함께 관리하고 싶어서 AuthenticationUtils.js 파일 이름을  AuthUtils.js로 바꿨다…ㅎㅎ*

```jsx
const authorizeBoardMiddleware = (req, res, next) => {
    const boardId = req.params.id;
    const board = boardController.findBoardById(boardId);
    if (board.user_id !== req.session.user.user_id) {
        return res.status(403).json(makeRes(403, "required_permission", null));
    }
    next();
}

const authorizeCommentMiddleware = (req, res, next) => {
    const commentId = req.params.commentId;
    const comment = commentController.findBoardById(commentId);
    if (comment.user_id !== req.session.user.user_id) {
        return res.status(403).json(makeRes(403, "required_permission", null));
    }
    next();
}
```

```jsx
// 게시글 수정 "/:id"
router.patch("/:id", authenticateMiddleware, authorizeBoardMiddleware, boardController.patchBoard);

// 게시글 삭제 "/:id"
router.delete("/:id", authenticateMiddleware, authorizeBoardMiddleware, boardController.deleteBoard);

// 댓글 수정 "/{post_id}/comments/{comment_id}" *
router.patch("/:postId/comments/:commentId", authenticateMiddleware, authorizeCommentMiddleware, commentController.patchComment);

// 댓글 삭제 "/{post_id}/comments/{comment_id}" *
router.delete("/:postId/comments/:commentId", authenticateMiddleware, authorizeCommentMiddleware, commentController.deleteComment);
```

위와 같이 인가 확인 미들웨어함수를 데이터 수정, 삭제 api에 추가해주었다. 

또한 프론트엔드에 403 상태코드가 올 때 `alert();` 를 주도록 수정했다.

```jsx
async function patchData(jsonData, path){
    const address = getBackendDomain() + path;
    try{
        const response = await fetch(address, {
            method: 'PATCH',
            credentials: 'include',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify(jsonData)
        });
        if (response.status === 401) {
            window.location.href = '/login';
        } else if (response.status === 403) {
            alert("권한이 없습니다!");
        }
        return response.json();
        }catch(error) {
        console.error('Error:', error);
    }
}
```

# 전체 코드 참고

[GitHub - Ssun2zang/5-haisely-express-be](https://github.com/Ssun2zang/5-haisely-express-be)

[GitHub - Ssun2zang/5-haisely-express-fe](https://github.com/Ssun2zang/5-haisely-express-fe)