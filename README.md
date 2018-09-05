Redux + CustomEventを使ったら実装が楽になった話
===

2018/09/05

Kobayashi Kazuhiro (kzhrk)

---

## TL;DR

### この記事で理解できるもの  
- Redux実装の最小構成 (公式サイトの[Basics](https://redux.js.org/basics))
- CustomEventによるイベント管理

### 書いていないもの  
- Reactとの連携
- 非同期のactionsなど (公式サイトの[Advanced](https://redux.js.org/advanced)に書いてあること)

---

## Reduxとは

状態管理のフレームワーク。  
Reactの状態管理で主に使われるが、他のフレームワークとの併用も可能。

![flowchart](https://user-images.githubusercontent.com/548508/45066232-65237280-b0f8-11e8-87a1-d2699071c4cb.png)

---

## Redux三原則

- アプリケーションのstateはひとつのstoreで管理される
- stateはread-onlyなもので、actionsを通すことでしか変更できない
- stateの変更は純粋関数*(reducers)によって定義される

*参考 [そのJavaScriptは純粋関数？](https://postd.cc/httpstaltz-comis-your-javascript-function-actually-pure/)

---

## Actions

dispatchに引き渡すObjectで、必ずtypeプロパティを持つ。  
actionsは”何が起きたか”という事実を説明するだけのもので、stateにどんな変化を与えるかを表現しない。

actionsが多いときはtypes.jsのようにモジュールを分割するとよい。

```js
import types from './types';

// actions
export const initCar = {
  type: types.INIT_CAR,
  cars: []
};

// action creaters
export const addCar = car => {
  return {
    type: types.ADD_CAR,
    car
  };
};

export const deleteCar = car => {
  return {
    type: types.DELETE_CAR,
    car
  };
};
```

```js
[...]
store.dispatch(actions.addCar('toyota'));
[...]
```

---

## Reducers

アプリケーションのstateに変更を加える関数。  
引数で渡されたactionsの値によってstateを更新する。

stateの初期化はここで行われる。  
UIのstateとは切り離し、アプリケーション内で保持する必要があるstateの構造を設計する。

reducersは第一引数にactionsが実行される前のstate、第二引数にdispatchされたactionsを受け取る。

reducers内では以下の3つの行為は禁止されている。  
- 引数の変更 (stateに新しいプロパティを追加するなど)
- API呼び出し
- 純粋関数ではない関数の呼び出し (例Date.now(), Math.random())

```js
import types from './types';

const INITIAL_STATE = {
  cars: []
};

const reducer = (state = INITIAL_STATE, action) => {
  switch (action.type) {
    case types.ADD_CAR: {
      const { car } = action;
      const { cars } = state;

      if (!cars.includes(car)) cars.push(car);

      return {
        ...state,
        cars
      };
    }
    case types.DELETE_CAR: {
      const { car } = action;
      const { cars } = state;

      const index = cars.indexOf(car);

      if (index !== -1) cars.splice(index, 1);

      return {
        ...state,
        cars
      };
    }
    default: {
      return state;
    }
  }
};

export default reducer;
```

### The switch statement is not the real boilerplate.

> 多くのアプリケーションはswitch文でreducersを定義しているが、switch文はactionsの数だけcaseブロックが増えるのであまり推奨されない。  
> https://redux.js.org/basics/reducers#note-on-switch-and-boilerplate

### createReducer

reducersに渡されたaction.typeが、引数となるObjectのプロパティと一致したときにその関数を実行する。  
処理内容はswitch文と全く同じ。

```js
import types from './types';

const INITIAL_STATE = {};

function createReducer (initialState, handlers) {
  return function reducer(state = initialState, action) {
    if (handlers.hasOwnProperty(action.type)) {
      return handlers[action.type](state, action);
    } else {
      return state;
    }
  }
}

const reducer = createReducer(INITIAL_STATE, {
  [types.ADD_CAR](state, action) {
    const { car } = action;
    const { cars } = state;

    if (!cars.includes(car)) cars.push(car);

    return {
      ...state,
      cars
    };
  },
  [types.DELETE_CAR](state, action) {
    const { car } = action;
    const { cars } = state;

    const index = cars.indexOf(car);

    if (index !== -1) cars.splice(index, 1);

    return {
      ...state,
      cars
    };    
  }
});

export default reducer;
```

reducers内の処理をObjectとして書けるので更新対象となるstateのカテゴリ毎にmodule分割することが可能になり、見通しがよくなりテストも書きやすくなる。

```js
import createReducer from 'createReducer';
import cars from './cars';
import models from './models';

const reducer = createReducer(INITIAL_STATE, {
  ...cars,
  ...models
});

export default reducer;
```

```js
// cars.js
import types from './types';

export {
  [types.ADD_CAR](state, action) {
    const { car } = action;
    const { cars } = state;

    if (!cars.includes(car)) cars.push(car);

    return {
      ...state,
      cars
    };
  },
  [types.DELETE_CAR](state, action) {
    const { car } = action;
    const { cars } = state;

    const index = cars.indexOf(car);

    if (index !== -1) cars.splice(index, 1);

    return {
      ...state,
      cars
    };    
  }
} 
```

```js
// cars.spec.js
import chai from 'chai';
import reducer from './reducers/cars';

describe('Cars', ()=> {
  it('車を追加すること', (done) => {
    [...]
  });
});
```

---

## Store

actionsとreducersで更新されたstateが集約されるObject。  
storeは下記の3つのメソッドを持つ。

- getState() stateを取得
- dispatch(action) stateの更新
- subscribe(listener) イベントの登録/削除

storeの定義は`createStore()`で行われる。  
第一引数にreducersを渡し、オプションとして第二引数に初期stateを渡すことができる。

```js
import { createStore } from 'redux';
import reducer from './reducers';

export default createStore(reducer, STATE_FROM_SERVER);
```

---

## Directory Structure

```
src
└── webpack
    └── store
        ├── actions.js
        ├── index.js
        ├── reducers.js
        └── types.js
```

---

## types.js

actionsとreducersで使用するtypeの定数。

```js
export default {
  SET_ACTIVE_TAB: 'SET_ACTIVE_TAB',
  SET_ACTIVE_SERIES: 'SET_ACTIVE_SERIES',
  SET_CURRENT_SERIES: 'SET_CURRENT_SERIES',
  SET_ACTIVE_MODEL: 'SET_ACTIVE_MODEL',
  SET_CURRENT_MODEL: 'SET_CURRENT_MODEL',
  SET_CURRENT_CAR: 'SET_CURRENT_CAR',
  DELETE_CAR: 'DELETE_CAR',
  ADD_CAR: 'ADD_CAR',
  TOGGLE_CAR: 'TOGGLE_CAR'
};
```

---

## actions.js

```js
import types from './types';

export const setActiveTab = tab => {
  return {
    type: types.SET_ACTIVE_TAB,
    tab
  };
};
[...]
```

---

## reducers.js

```js
import types from './types';

const INITIAL_STATE = {
  activeTab: '',
  activeSeries: '',
  currentSeries: '',
  activeModel: '',
  currentModel: '',
  currentCar: '',
  cars: []
};

function createReducer (initialState, handlers) {
  return function reducer(state = initialState, action) {
    if (handlers.hasOwnProperty(action.type)) {
      return handlers[action.type](state, action);
    } else {
      return state;
    }
  }
}

export default const reducer = createReducer(INITIAL_STATE, {
  [types.SET_ACTIVE_TAB] (state, action) {
    const { tab } = action;

    return {
      ...state,
      activeTab: tab
    };
  },
  [...]
};
```

---

## index.js

```js
import { createStore } from 'redux';
import reducer from './reducers';

export default createStore(reducer);
```

---

## CustomEvent

イベントは`Event`から作成が可能。  
たとえば、checkboxのchangeイベントを定義後、checkboxをcheckedにしてからchangeイベントを発火するときは以下のようになる。

```js
this.checkboxes = document.querySelectorAll('input[type="checkbox"]');

[...this.checkboxes].forEach(checkbox => {
  checkbox.addEventListener('change', this.handleChangeCheckbox.bind(this), false);
});

const changeEvent = new Event('change');

[...this.checkboxes].forEach(checkbox => {
  checkbox.checked = true;
  checkbox.dispatchEvent(changeEvent);
});
```

`CustomEvent`からカスタムイベントを作成するとイベントオブジェクトへのデータ追加、イベント伝播のハンドリングを行うことが可能。

```js
const root = document.body;
const tabs = document.querySelectorAll('.js-tab');
const clickTab = new CustomEvent('clickTab', {
  bubbles: true
});

root.addEventListener('clickTab', () => {
  console.log('clicked tab');
}, false);

[...tabs].forEach(tab => {
  tab.addEventListener('click', () => {
    root.dispatchEvent('clickTab');
  }, false);
});
```

---

## CustomEvent Polyfill

CustomEventはIE未サポートなので、Pollyfillを使用する。  
webpackのresolve.aliasに設定するとPolyfillのimportもなく実装ができてよい。  
[CustomEvent() - Web APIs | MDN](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent/CustomEvent)

```js
let CustomEvent;

if (typeof window !== 'undefined' && typeof window.CustomEvent !== 'function') {
  CustomEvent = function(
    event,
    params = { bubbles: false, cancelable: false, detail: undefined }
  ) {
    let evt = document.createEvent('CustomEvent');
    evt.initCustomEvent(
      event,
      params.bubbles,
      params.cancelable,
      params.detail
    );
    return evt;
  };

  CustomEvent.prototype = window.Event.prototype;
} else {
  CustomEvent = window.CustomEvent;
}

export default CustomEvent;
```

webpack.config.js  
```js
  [...]
  resolve: {
    extensions: ['.js'],
    alias: {
      CustomEvent: path.resolve(
        __dirname,
        './src/webpack/helpers/polyfill-custom-event.js'
      )
    }
  },
  [...]
```

---

## CustomEventの定義

### customEvents.js

```js
export const eventName = {
  clickTab: 'clickTab'
};

const clickTab = new CustomEvent(eventName.clickTab, {
  bubbles: true
});

export const event = {
  clickTab
};
```

---

### webpackのentry pointになるJS

```js
import { eventName } from './modules/customEvents';

import Tab from './modules/Tab';

window.addEventListener(
  'load',
  () => {
    const el = document.getElementById('js-select');

    const tab = new Tab({ el });

    // tab events
    el.addEventListener(
      eventName.clickTab,
      () => {
        tab.changeActive();
      },
      false
    );
  },
  false
);
```

Tabモジュール

```js
import store from '../store';
import actions from '../store/actions';
import { event } from './customEvents';

const className = {
  active: 'is-active'
};

export default class Tab {
  constructor({
    el,
    tabs = el.querySelectorAll('.js-tab'),
    contents = el.querySelectorAll('.js-tab-contents')
  }) {
    this.el = el;
    this.tabs = tabs;
    this.contents = contents;

    // store初期化
    store.dispatch(actions.setActiveTab(this.tabs[0].dataset.target));

    this.changeActive();
    this.addEvent();
  }
  addEvent() {
    [...this.tabs].forEach(tab => {
      tab.addEventListener(
        'click',
        this.handleClickTab.bind(this),
        false
      );
    });
  }
  handleClickTab(e) {
    const tab = e.currentTarget;

    store.dispatch(actions.setActiveTab(tab.dataset.target));

    // seriesとmodelsのカレント更新
    store.dispatch(actions.setCurrentSeries(store.getState().activeSeries));

    store.dispatch(actions.setCurrentModel(store.getState().activeModel));

    this.el.dispatchEvent(event.clickTab);
  }
  changeActive() {
    // toggle active class
    [...this.tabs].forEach(tab => {
      tab.dataset.target === store.getState().activeTab
        ? tab.classList.add(className.active)
        : tab.classList.remove(className.active);
    });

    // toggle contents
    [...this.contents].forEach(content => {
      content.dataset.target === store.getState().activeTab
        ? (content.style.display = 'block')
        : (content.style.display = 'none');
    });
  }
}
```

---

## redux + CustomEventで実装した所感

reduxにページ内のstateを持たせることで、module間でネーム変数参照を行う必要がなくなってmoduleの疎結合が実現できた。  
CustomEventでイベント処理をentry pointにまとめることでユーザのアクションのハンドリングの一覧性が向上した。

- module内ではreduxのstoreの更新、dispatchEventでルートのDOMにイベントを伝える  
- moduleにstoreの参照をしてDOMを操作するメソッドを作成し、entry pointから実行する

