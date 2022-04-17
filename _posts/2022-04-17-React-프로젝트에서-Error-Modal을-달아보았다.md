---
layout: post
title:  "React 프로젝트에서 Error Modal을 달아보았다"
excerpt: "React 프로젝트에서 Error Modal을 달아보았다. 그런데 Redux를 이용한 상태관리를 곁들인."
date: "2022-04-17"

author: zoooo-hs
tags: [React, Redux, Frontend, Web]

---

* TOC
{:toc}

[게시글에 사용된 소스코드](https://github.com/zoooo-hs/zoooo-hs.github.io-source-codes/tree/main/2022-04-17-react-error-modal)

# 서론 - 기존에 모달을 추가한 방법

프론트엔드 프로젝트를 진행하다 보면 API 호출 결과에 대한 오류를 사용자에게 보여줘야 할 경우가 생긴다. [인스타그램 클론 토이프로젝트](https://github.com/zoooo-hs/instagra-clone-fe)를 진행하면서 해당 기능이 필요로 했는데, 이를 모달로 보여주고자 했다. 기존에 모달을 만들어 다른 기능들을 만들어둔 상태라 모달을 만드는 것은 그리 어렵지 않았다. 아래와 같은 순서로 모달을 붙일 수 있었는데,

- 모달을 만들 컴포넌트에 모달을 열지 말지를 결정하는 boolean state를 생성
- 컴포넌트 render 부분 (함수형의 경우 return)에 모달 상태에 맞춰 모달 컴포넌트를 표시
- 모달 컴포넌트의 특정 액션으로 모달 창을 닫고자 한다면, 함수를 prop으로 넘겨 모달 상태 값을 바꿀 수 있게 함

```tsx
...
const [isOpened, setOpened] = useState(false);
...

return (
  ...
    {isOpened ? <MyModal ... callback={() => {setOpened(false);}} /> : null}
  ...
);
```

이러한 방법으로 모달을 붙일 수 있지만, 이번에 오류 결과를 보여주는 모달의 경우 위처럼 작성할 시 한 가지 이슈가 발생한다. 오류는 API 호출 결과를 통해 알 수 있고, 오류는 여러 컴포넌트에서 발생할 수 있다. 그러므로 모달을 여닫는 코드가 여러 컴포넌트에 중복으로 작성될 수 있다. 이러면 오류 모달에 대한 코드가 일정부분 변경 되었을 때 일괄 갱신하기가 쉽지 않다. 백엔드에서 전달해주는 오류에 대한 데이터 모델은 정해져 있기 때문에 전역에서 접근할 수 있는 모달 컴포넌트가 있다면 해결될 문제이다. 이런 부분은 redux와 같은 상태 관리를 이용해 해결 할 수 있을 것 같았고, 구글링해본 결과 여러 글<sup>[1](#footnote_1),[2](#footnote_2)</sup>이 있어 종합하여 내 코드를 작성해 보았다.

## 들어가기 앞서: 사용한 기술 스택

- React + Redux
- Typescript

# 코드 작성 시작

## 적용할 방법의 간략한 흐름

코드로 설명하기에 앞서 어떤 흐름으로 코드를 구성할지 아래에 기술하였다.

- 표현하고자 하는 오류의 모델 인터페이스를 작성
- 오류 데이터 및 모달 표시 유무 상태 관리를 위한 state, action, reducer 작성
- 오류 상태에 따라 화면에 표시할 모달 컴포넌트 작성
- 오류 발생에 따라 오류 상태 변화 액션 호출

위 흐름대로 코드를 작성해볼 텐데, 각자 프로젝트와 API 데이터 상황에 맞춰 알맞은 필드 및 상태를 만들어 사용하면 될 것이다.

## 오류 모델 인터페이스 작성

오류를 단순히 문자열로 변경하여 오류 모달에 문자열을 그대로 보여줘도 상관없지만, 앱에서 보여주는 오류가 정형화된 데이터 모델로 표현할 수 있다면 해당 오류를 인터페이스로 정의하자. 본 글에서는 백엔드 API 호출로 반환되는 오류를 주로 보여주고, 다음과 같은 형식으로 오류가 구성되어있다고 가정한다.

```tsx
// src/model/error-model.ts

export interface ErrorModel {
    code: string, // USER_NOT_FOUND, INVALID_LOGIN_FORM 등, 오류의 식별자
    message: string // 오류 메시지
}
```

## 오류 데이터 상태를 관리할 action, reducer 작성

앱에서 오류를 보여줄 것인지 그리고 현재 보여줄 오류가 어떤 오류인지를 앱의 글로벌 상태로 기록한다. 본 글에서는 앱의 상태관리를 위해 redux를 사용했다.

redux 사용 방법은 redux 문서<sup>[3](#footnote_3)</sup>를 참고했다.

### 앱에서 관리할 상태

오류 모달을 사용하기 위해 앱 전역에서 참조하고 관리할 상태는 다음과 같다.

- 오류 데이터
    - ErrorModel 타입
    - 모달에 출력할 오류 데이터
- 모달 표시 여부
    - boolean 타입
    - 모달을 앱 화면에 표시할지 말지 결정

이러한 요소를 갖춘 interface를 정의하고, reducer에서 사용할 초기 상태 상수를 정의한다. 초기에 아무런 에러가 없을 수 있으므로 error는 optional로 정의한다.

```tsx
interface ErrorState {
    error?: ErrorModel;
    visible: boolean;
}

export const initErrorState: ErrorState = { error: undefined, visible: false };
```

### 앱에서 상태를 변경하기 위한 액션 정의

앱에서 오류 상태를 변경하는 상황과 필요한 액션은 다음과 같다.

- SHOW_ERROR: 오류가 발생하여 오류 상태를 새로운 오류로 갱신하고, 오류 모달을 보여줌
    - ErrorState.error를 새로운 오류로 갱신
    - ErrorState.visible을 true로 갱신
- HIDE_ERROR: 오류 모달을 닫아 오류 모달을 숨기고, 오류 정보를 제거
    - ErrorState.error를 undefined로 갱신
    - ErrorState.visible을 false로 갱신

두 액션을 다음과 같이 정의한다.

```tsx
interface Action { // Reducer에서 각 Action에서 넘겨준 데이터를 참조할때 사용할 데이터 타입 정의
    type: string,
    payload: ErrorState
}

export const SHOW_ERROR = "ERROR_STATE/SHOW_ERROR";
export const HIDE_ERROR = "ERROR_STATE/HIDE_ERROR";

export function showError(error: ErrorModel): Action {
    return {
        type: SHOW_ERROR,
        payload: {
            error,
            visible: true
        }
    }
}

export function hideError(): Action {
    return {
        type: HIDE_ERROR,
        payload: {
            error: undefined,
            visible: false
        }
    }
}
```

### 액션을 받아 처리할 리듀서 정의

앞서 정의한 액션들을 액션 타입에 맞춰 상태 변경할 리듀서를 다음과 같이 정의한다.

```tsx
export const errorState = (state: ErrorState = initErrorState, action: Action): ErrorState => {
    switch (action.type) {
        case SHOW_ERROR:
        case HIDE_ERROR:
            return { ...state, ...action.payload }
        default:
            return state;
    }
}
```

이후 reducer를 store에 담아 export 하여 각 컴포넌트에서 사용한다.

```tsx
// src/reducer/index.ts

import {combineReducers, createStore} from "redux";
import {errorState} from "./error-state";

const rootReducer = combineReducers({
    errorState
})

export default rootReducer;

export type RootState = ReturnType<typeof rootReducer>
export const store = createStore(rootReducer);

// src/index.tsx

...
import { Provider } from 'react-redux';
import { store } from './reducer';

...
root.render(
  <Provider store={store}>
      ...
      <App />
      ...
  </Provider>
);
...
```

## 오류 내용을 보여줄 ErrorModal 컴포넌트 정의

이제 오류 상태에 따라 화면에 표시할 모달 컴포넌트를 정의한다. 컴포넌트는 error 및 표시 여부와 관련된 prop을 따로 요구하지 않고, 앞서 정의한 상태를 직접 받아와 사용한다. 

이러한 ErrorModal 컴포넌트의 구성 및 동작은 다음과 같다.

- ErrorState를 불러온다.
- ErrorState.visible, ErrorState.error 정보에 따라 모달을 표시하지 않으면 null을 반환
    - visible === false OR error === undefined
- 모달을 표시할 때 ErrorState.error에 있는 정보를 render 하며 이때 button을 하나 두어 모달을 닫을 수 있도록 한다.
    - 해당 버튼의 onClick 핸들러 함수에서 HIDE_ERROR 액션을 dispatcher에 전달한다.

```tsx
// src/component/error-modal.tsx

import { useDispatch, useSelector } from "react-redux";
import { RootState } from "../reducer";
import { hideError } from "../reducer/error-state";
import "./error-modal.css";

export function ErrorModal() {
    const {error, visible} = useSelector((state: RootState) => state.errorState);
    const dispatch = useDispatch();

    if (!visible || error === undefined) {
        return null;
    } 

    function handleClose() {
        dispatch(hideError());
    }

    const {code, message} = error;

    return (
        <div className="error-modal">
            <h1>{`[오류 코드 : ${code}] 오류 발생`}</h1>
            <p>{message}</p>
            <button onClick={handleClose}>닫기</button>
        </div>
    )
}
```

이후 ErrorModal을 App 컴포넌트의 자식 컴포넌트로 추가한다.

```tsx
// src/App.tsx

import { ErrorModal } from "./component/error-modal";

function App() {
  return (
    <div>
      <ErrorModal/>
    </div>
  );
}

export default App;
```

## 오류가 발생할 경우 상태 변화 액션 호출

위 내용까지 모두 작성했다면, 이제 오류가 발생할 때 SHOW_ERROR 액션을 통해 상태를 변경하면 된다. 본 글에서는 특별한 상황을 가정하여 오류를 발생시키고 상태 변화 및 모달이 잘 작동하는지를 확인한다. 

### 테스트 - 매번 오류를 발생시키는 버튼

작성한 상태 관리 및 모달 컴포넌트가 잘 작동하는지 확인하기 위해 App 컴포넌트에 임의의 버튼을 만들고 버튼의 onClick 핸들러는 매번 새로운 오류가 발생하는 API를 호출한다고 가정하자. 이러한 오류가 매번 발생할 때마다 오류 모달이 잘 갱신되는지 확인한다.

먼저 버튼 클릭시 매번 오류를 생성할 함수 troubleMaker를 작성한다. 오류 메시지는 현재 시각을 담고 있다.

```tsx
function troubleMaker(): ErrorModel {
  return {
    code: "ERROR_0001",
    message: `새로운 오류! 현재 시간은 ${new Date()}`
  }
}
```

이후 App 컴포넌트에 button을 추가하고 onClick 핸들러는 troubleMaker를 호출하여 오류가 발생하면 SHOW_ERROR 액션을 dispatcher에 전달한다.

```tsx
function App() {

  const dispatch = useDispatch();

  function handleClick() {
    troubleMaker().then(() => {
      // never
    }).catch(error => {
      dispatch(showError(error));
    });
  }

  return (
    <div>
      <ErrorModal/>
      <button onClick={handleClick}>트러블 메이커</button>
    </div>
  );
}
```

![visible false](/assets/img/20220417/error-1.png)

![visible true](/assets/img/20220417/error-2.png)

의도한 대로 상태변화를 통해 모달이 보여지고 사라지는 것을 확인 할 수 있다.

### 부록: Axios interceptor

위와 같은 단편적인 예시 말고 조금 더 실용적인 예시를 찾아본다면, Axios로 HTTP 호출을 할 때 interceptor를 통해 오류 응답에 대해서 모달 상태변화 액션을 호출할 수 있다. 

```tsx
...
axios.interceptors.response.use(response => response, async (error) => {
  dispatch(showError(error));
  return Promise.reject(error);
});
...
```

# 끝

이번 글에서는 React 프로젝트에서 오류 모달을 작성하는 방법에 관해 기술해보았다. 위 방법을 통해 오류 모달을 보여줘야 할 컴포넌트마다 따로 모달을 위한 상태 관리 및 모달 컴포넌트 추가를 방지할 수 있어 중복 코드가 많이 줄어든다.


# 참고 자료

<ol>
  <li><a name="footnote_1">https://andremonteiro.pt/react-redux-modal/</a></li>
  <li><a name="footnote_2">https://www.pluralsight.com/guides/centralized-error-handing-with-react-and-redux</a></li>
  <li><a name="footnote_3">https://react-redux.js.org/using-react-redux/usage-with-typescript</a></li>
</ol>
