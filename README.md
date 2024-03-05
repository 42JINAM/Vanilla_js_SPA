# Vanilla_js_SPA
## Router
```js
class Router {
  #routes;
  #errorPage;
  #view;
  constructor() {
    this.#errorPage = {path: '/404', view: 'ErrorPage'};
    this.#routes = [
      {path: '/', view: 'Home'},
      {path: '/user/:intra_id', view: 'User'},
      {path: '/game', view: 'Game'},
      {path: '/rank', view: 'Rank'},
      {path: '/login', view: 'Login'},
      {path: '/auth/ft/redirection', view: 'Auth'},
      {path: '/error/:code', view: 'ErrorPage'},
    ];
  }
  pathToRegex(path) {
    return new RegExp(
        '^' + path.replace(/\//g, '\\/').replace(/:\w+/g, '(.+)') + '$',
    );
  }
  getParams(match) {
    const values = match.result.slice(1);
    const keys = Array.from(match.route.path.matchAll(/:(\w+)/g)).map(
        (result) => result[1],
    );
    return Object.fromEntries(
        keys.map((key, i) => {
          return [key, values[i]];
        }),
    );
  }

  async navigateTo(url, title = null) {
    if (url != undefined && url !== location.href) {
      history.pushState({}, title, url);
    } else {
      history.replaceState({}, title, url);
    }
    const view = await this.router();
    return view;
  }

  async backNavi() {
    if (this.#view != null) this.#view.unmounted();
    const view = await this.router();
    return view;
  }

  async router() {
    if (this.#view != null) {
      this.#view.unmounted();
      if (this.#view._title == 'Game') this.#view.game_unmounted();
    }
    const potentialMatches = this.#routes.map((route) => {
      return {
        route: route,
        result: location.pathname.match(this.pathToRegex(route.path)),
      };
    });
    let match = potentialMatches.find(
        (potentialMatches) => potentialMatches.result !== null,
    );
    if (!match) {
      match = {
        route: this.#errorPage,
        result: [location.pathname],
      };
    }
    const params = this.getParams(match);
    if (match.route.view === 'User' &&
      await Utils.isValidIntra(params.intra_id) === false) {
      this.#view =
        await import(`../components/pages/${this.#errorPage.view}.js`)
            .then(async ({default: Page}) => {
              return new Page(document.querySelector('#app'), {code: 404});
            });
      return this.#view;
    }
    this.#view = await import(`../components/pages/${match.route.view}.js`)
        .then(async ({default: Page}) => {
          return new Page(document.querySelector('#app'), params);
        });
    return this.#view;
  }
}

export default new Router();

```


## Components 
### component
``` js
import {observable, observe} from './observer.js';
import * as Lang from '../Lang.js';
import {authReq} from '../connect.js';

export default class Component {
  /**
   * @type {Element} $target
   */
  $target;
  props;
  state = {};
  constructor(parent, $target) {
    this.id = '10000000-1000-4000-8000-100000000000'.replace(/[018]/g, (c) =>
      (c ^ crypto.getRandomValues(new Uint8Array(1))[0] & 15 >> c / 4)
          .toString(16),
    );
    this.$target = $target;
    this.parent = parent;
    this.events = [];
    this.children = [];
    this.onexception = (e) => {
      if (this.parent) this.parent.onexception(e);
      else this.unmounted();
    };
    this.setup();
  }

  async setup() {
    this.state = observable(await this.initState());
    observe(async () => {
      try {
        this.unmounted();
        await this.render();
        this.setEvent();
        await this.mounted();
      } catch(e) {
        if (e instanceof NoError) {
          // do nothing
        }
        this.onexception(e);
      }
    }, this.state);
  }

  async initState() {
    return {};
  }
  async mounted() {}

  async template() {
    return '';
  }

  async render() {
    this.$target.innerHTML = await this.template();
    Lang.loadLanguage();
  }

  setEvent() {}

  clearEvent() {
    for (const event of this.events) {
      this.$target.removeEventListener(event.eventType, event.callback);
    }
    this.events = [];
  }

  addEvent(eventType, selector, callback) {
    const e = (event) => {
      if (!event.target.closest(selector)) return false;
      callback(event);
    };
    this.events.push({eventType, callback: e});
    this.$target.addEventListener(eventType, e);
  }

  addComponent(component) {
    this.children.push(component);
  }

  popComponent() {
    const component = this.children.pop();
    if (component) component.unmounted();
  }

  unmounted() {
    for (const child of this.children) {
      child.unmounted();
    }
    this.clearEvent();
    this.children = [];
  }
}

Component.prototype.authReq = authReq;

export class NoError extends Error {
}
```
### observer 
``` js
let currentObserver = null;

function debounceFrame(callback) {
  let currentCallback = -1;
  return () => {
    cancelAnimationFrame(currentCallback);
    currentCallback = requestAnimationFrame(callback);
  };
}

export async function observe(fn, obj) {
  currentObserver = debounceFrame(fn);
  for (const key in obj) {
    if (Object.hasOwn(obj, key) == false) continue;
    obj[key];
  }
  await fn();
  currentObserver = null;
}

export function observable(obj) {
  const observerMap = Object.keys(obj).reduce((map, key) => {
    map[key] = new Set();
    return map;
  }, {});
  return new Proxy(obj, {
    get: (target, name) => {
      observerMap[name] = observerMap[name] || new Set();
      if (currentObserver) observerMap[name].add(currentObserver);
      return target[name];
    },
    set: (target, name, value) => {
      if (target[name] === value) return true;
      if (JSON.stringify(target[name]) === JSON.stringify(value)) return true;
      target[name] = value;
      observerMap[name].forEach((fn) => fn());
      return true;
    },
  });
}
```
## Api Handling with jwt
``` js
function apiServerEndPoint(uri) {
  return `${pubEnv.API_SERVER}${uri}`;
}

async function request(uri, options) {
  let res;
  let json;
  try {
    res = await fetch(uri, options);
    json = await res.json();
    return [res, json];
  } catch (e) {
    throw new RequestError(e, res.status);
  }
};

export function parseJwt(token) {
  const base64Url = token.split('.')[1];
  const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
  const payload = decodeURIComponent(window.atob(base64).split('').map((c) => {
    return '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2);
  }).join(''));
  return JSON.parse(payload);
};

async function refreshToken() {
  const refresh = Cookie.getCookie(pubEnv.TOKEN_REFRESH);
  const access = Cookie.getCookie(pubEnv.TOKEN_ACCESS);
  if (refresh == null) throw new RequireLoginError();
  try {
    const res = await fetch(apiServerEndPoint('/api/token/verify/'), {
      method: 'post',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        access,
        refresh,
      }),
    });
    if (res.status === 201) Cookie.setToken(await res.json());
    else if (res.status === 200) return;
    else throw new IssueTokenError();
  } catch (e) {
    throw new IssueTokenError(e);
  }
}

export async function authReq(method, uri, body = {}) {
  try {
    if (method == null) throw new Error();
    method = method.toUpperCase();
    const endpoint = apiServerEndPoint(uri);
    let access = Cookie.getCookie(pubEnv.TOKEN_ACCESS);
    if (access == null) throw new RequireLoginError();
    const req = {
      method,
      headers: {'Authorization': `Bearer ${access}`},
    };
    if (method !== 'GET') {
      req.body = JSON.stringify(body);
      req.headers['Content-Type'] = 'application/json';
    }
    const expiredAt = parseJwt(access)['exp'];
    if (expiredAt == null) throw new Error();
    if (Date.now() < (expiredAt * 1000) - 10000) {
      const [res, json] = await request(endpoint, req);
      if (res.status != 401) return [res, json];
    }
    await refreshToken();
    access = Cookie.getCookie(pubEnv.TOKEN_ACCESS);
    const ret = await request(endpoint, {
      ...req,
      headers: {...req.headers, 'Authorization': `Bearer ${access}`},
    });
    return ret;
  } catch (e) {
    if (e instanceof RequireLoginError || e instanceof IssueTokenError) {
      Cookie.deleteCookie(
          pubEnv.TOKEN_ACCESS,
          pubEnv.TOKEN_REFRESH,
          pubEnv.TOKEN_INTRA_ID,
      );
      Router.navigateTo('/');
    } else {
      Router.navigateTo(`/error/${e.code}`);
    }
    throw e;
  }
}

export class IssueTokenError extends Error {
  constructor(e) {
    super();
    this.type = 'issueTokenError';
    this.e = e;
  }
}

export class RequireLoginError extends Error {
  constructor(e) {
    super();
    this.type = 'requireLoginError';
    this.e = e;
  }
}

export class RequestError extends Error {
  constructor(e, code) {
    super();
    this.type = 'requestError';
    this.code = code;
    this.e = e;
  }
}
```

## Example : FT_transendence


<img width="2560" src="https://github.com/42JINAM/Vanilla_js_SPA/assets/158562186/774347de-1af0-405c-9a30-3ebe1d335444"/>



###  User page
<img width="2545" alt="Screen Shot 2024-03-05 at 7 15 39 PM" src="https://github.com/42JINAM/Vanilla_js_SPA/assets/158562186/d2f2fc2f-d47d-41ae-bde7-16719e670a1f">

#### Profile
#### My Profile

https://github.com/42JINAM/Vanilla_js_SPA/assets/158562186/2da2c6e8-0e48-4040-95cc-06c0f6f23788

#### History

<img width="2560" alt="Screen Shot 2024-03-05 at 8 00 56 PM" src="https://github.com/42JINAM/Vanilla_js_SPA/assets/158562186/83b618ba-de10-4f47-bb96-3a4f29795888">


### Game page

<img width="2560" src="https://github.com/42JINAM/Vanilla_js_SPA/assets/158562186/45bd3c7e-756e-432e-9c17-2f671f8f9d99"/>


<img width="2560" src= "https://github.com/42JINAM/Vanilla_js_SPA/assets/158562186/2f141266-bfca-4e0c-8833-38e6000a0261"/>
<img width="2560" src= "https://github.com/42JINAM/Vanilla_js_SPA/assets/158562186/cb657615-45d3-4c05-9873-0db95f097437"/>


