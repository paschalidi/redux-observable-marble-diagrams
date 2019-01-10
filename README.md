Marble diagrams in redux-observable
In this tutorial, I’ll show you a very interesting way to test epics in redux-observable: by using marble diagrams. This testing method has become possible only recently in version 1 of redux-observable. Although v1 is still technically in alpha, the library itself is quite stable and has been around since April 2016. I’ve used redux-observable in production for nearly a year with no trouble.

The end result of this tutorial is a very simple web app that fetches and displays the content of GitHub file URLs. These URLs are what you see in the browser’s address bar when browsing a file in GitHub’s web UI, for example, https://github.com/ReactiveX/rxjs/blob/master/index.js. You can play with the app here. The redux-observable epic we’ll write will be responsible for fetching data from GitHub when user input changes. The code of this tutorial is contained in this repo.

This tutorial does assume that you have a basic understanding of Observables and RxJS. It also assumes that you have read redux-observable’s (very short) documentation. Here are the technical highlights:

Use TypeScript in production code and modern JavaScript in test code.
Use the latest version of RxJS (v6) with pipeable operators.
Use the latest version of redux-observable (v1.0.0-alpha.2) which allows consuming the state as an Observable of state changes.
Demonstrate unit testing of epics in redux-observable using marble diagrams in Jest.
Quick recap of redux-observable
The nature of side effects runs counter to the purity and memoryless-ness of reducer functions in redux. However, because no real-world applications can work without side effects, we have to find a way to make them work with redux. The main idea of all side-effect management approaches in redux is to execute functions that have access to the redux store’s dispatch method. These functions can orchestrate and dispatch their own sequence of actions to the store. However, each approach dresses up the function execution and the inherent statefulness of side effects (e.g. waiting for an API call to finish before dispatching the “done” action to the store) in a different way. There are currently many ways to do this, three of which I’ll summarize here.

The imperative solution is thunks, implemented by the redux-thunk middleware. Thunks are just plain old functions that receive the store’s dispatch function as a parameter. When the store receives a thunk, the store executes it, which uses dispatch to create its own sequence of actions. Although thunks are the easiest of the three to understand, unit testing thunks is tricky because it involves setting up an entire redux store. You can read more about the shortcomings of redux-thunk here.

In contrast, the redux-saga middleware is a declarative solution in which the application dispatches plain-object actions instead of functions. Statefulness is contained inside generator functions called sagas, which are executed when agreed-upon actions are dispatched from the application code. Side effects are described by effects, which are plain objects that contain instructions to the middleware to perform certain tasks. For example, suppose that the side effect is fetching from an API by calling fetchFromServer(a, b, c). When the application dispatches, say, {type: 'Fetch'} actions, the saga will run and emit an “function invocation” effect like this: yield call(fetchFromServer, a, b, c), causing the middleware to execute fetchFromServer(a, b, c). Unit testing a saga is fairly easy because it only involves asserting on the effects emitted by that saga and doesn’t require setting up a redux store.

The redux-observable middleware is another declarative solution. Like redux-saga, it uses plain-object actions. However, unlike redux-saga, to manage statefulness, redux-observable uses Observables, which are really just object wrappers around functions that take no arguments but “return” many values. redux-observable treats actions (dispatched by the application) and redux state changes as streams of events over time. Developers create a root epic, which is a single Observable that can in turn contain many child epics via composition, to transform and combine these streams using the machinery of RxJS into a single final stream of desired actions that will be dispatched to the store. If you think of the actions and state changes as streams of water then the root epic is a system of pipes that splits, channels and combines those streams to produce a single output stream. redux-observable is the perfect library to coordinate complex sequences of events that may overlap while avoiding race conditions. However, prior to v1 of redux-observable, unit testing epics without setting up a redux store was quite difficult, as shown by this long running GitHub issue. Furthermore, because epics represent streams of events over time, it can be difficult for beginners to instinctively understand how they work. This has changed for the better with the v1 release.

Pipeable operators in RxJS 6
The most important change from from version 5 to version 6 of RxJS is the replacement of instance-based operators with pipeable operators, which I think was a great decision by their team. The practical effect of that change on this tutorial is that if you’ve used RxJS 5 before and are accustomed to using instance-based operators (i.e. available on each Observable instance) like this:

const output$ = source$.filter(x => x > 10);
just know that they have been replaced by pipeable counterparts that need to be imported from rxjs/operators:

import {
  filter,
} from 'rxjs/operators'

const output$ = source$.pipe(
  filter(x => x > 10)
);
Marble diagrams
Marble diagrams are spatial representations of temporal event streams in RxJS. They are probably the most intuitive way to visualize RxJS operators. For example, this interactive marble diagram on rxmarbles.com is the graphical representation of how the filter operator works.

Graphical marble diagram of filter operator

In marble diagrams, each observable is represented by a timeline in which time flows from left to right. Each circle, called a marble, is an event that can optionally be associated with a value e.g. 2, 30, 22 etc in that diagram. Looking at the resulting Observable (bottom timeline), we can see that the effect of the filter operator is to remove values less than or equal to 10 from the source Observable (top timeline).

Because these diagrams are so easy to understand, they are great for visually testing whether epics behave in the intended manner. However, because we can’t use those colorful pictures in tests, there’s an equivalent text-based syntax. Although this syntax can represent many types of events, we’ll only use a few in this tutorial:

Within a marble string, whitespaces have no significance within marble diagrams. They are used mostly to vertically align diagrams for ease of viewing.
A dash '-' represents one “frame,” which is the unit of time in marble tests.
An alphanumeric character ([a-zA-Z0-9]) represents a normal event. The value of this event can either be the actual character itself or some other value if a mapping of the character to that other value is provided. Each character is considered emited at the start of the frame whose position is occupied by that character. Except for synchronous grouping (next bullet point), if a frame is occupied by a character, that frame cannot accommodate any other character.
Parentheses '()' represents synchronous grouping of events. All events enclosed within a pair of parentheses are considered emitted at the same time as the opening parenthesis.
A pipe | represents the successful completion of an Observable.
A hash sign '#' represents an error that terminates an Observable. The error can also be associated with a value (shown later).
Let’s try to recreate the colorful marble diagrams of filter above using the text-based syntax in a unit test. Because RxJS only provides marble testing functionality for Jasmine, we’ll use the excellent rxjs-marbles library, which provides a wrapper so that we can test marble diagrams in Jest:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/98861df6/src/__tests__/basic.js
 */
import {
  marbles,
} from 'rxjs-marbles/jest';
import {
  filter,
} from 'rxjs/operators';

it('filter operator', marbles(m => {
  const values = {
    a: 2,
    b: 30,
    c: 22,
    d: 5,
    e: 60,
    f: 1,
  };
  const source = m.cold('  -a-b-c-d-e-f---|', values);
  const expected = m.cold('---b-c---e-----|', values);
  const actual = source.pipe(
    filter(x => x > 10),
  );
  m.expect(actual).toBeObservable(expected);
}));
A few points to note in this test:

We wrap our test inside the marbles function provided by rxjs-marbles. marbles provides us with access to the functionality of RxJS’s built-in TestScheduler, which allows us to create a “cold” Observable with m.cold() and to assert that the resulting Observable matches our expectation with m.expect(...).toBeObservable(...).
We add two spaces at the beginning of source’s marble string so that it aligns with expected’s marble string.
We use | to mark the end of both Observables.
We associate the marbles a, b, c etc with numeric values 2, 30, 22 etc by passing a mapping (values) from the marble character to their corresponding values to the observable creator (m.cold). It is these numeric values that are passed to the predicate function inside the filter operator.
At this point, the repo shoud look like commit 98861df6. If you run yarn run test, the marble test should pass.

To see more marble diagrams, check out commit 6f944998 where I’ve written marble tests for most RxJS operators used in this tutorial. Unlike the diagram in the test for filter, the diagrams in these additional tests are not intended to replicate those on rxmarbles.com. A few things to note from these additional tests:

In the test for map, we didn’t provide any values for either the letters (a, b, c) so their emitted values are the actual letters or numbers themselves:
/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/6f944998/src/__tests__/basic.js
 */
// ...
  const source = m.cold('  -a-b-c--|');
  const expected = m.cold('-A-B-C--|');
  const actual = source.pipe(
    map(x => x.toUpperCase())
  );
  // ...
The tests for switchMap and concat show how whitespaces are very handy for visualizing temporal alignment of Observables. It’s important to remember that each Observable doesn’t start emitting until the first non-whitespace character in its marble diagram:
/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/6f944998/src/__tests__/basic.js
 */
// ...
  const source1 = m.cold(' --a--b--|');
  const source2 = m.cold('         c---d---|');
  const expected = m.cold('--a--b--c---d---|');
  const actual = concat(source1, source2);
  // ...
The test for debounceTime demonstrates how to properly marble test time-related operators. Each frame in the marble string corresponds to one “time unit” (whatever that unit actually is). In this test, we want to ensure that the source observable is debounced so that the resulting Observable doesn’t emit any event unless the source Observable has been quiet for 4 frames. Note that because timing in a marble test is controled by RxJS’s own TesScheduler (exposed as m.scheduler by the marbles wrapper), we need to pass it to the debounceTime operator. Without this custom scheduler, debounceTime will use the regular scheduler and actually debounce for 4 milliseconds, surely causing our test to fail.
/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/6f944998/src/__tests__/basic.js
 */
// ...
  const source = m.cold('  -a-a-a-a---------|');
  const expected = m.cold('-----------a-----|');
  const actual = source.pipe(
    debounceTime(4, m.scheduler)
  );
  // ...
The test for the catchError operator shows how to test errors in marble diagrams with the third argument to m.cold.
/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/6f944998/src/__tests__/basic.js
 */
// ...
  const errorMessage = 'This is an error!';
  const error = {
    name: 'Error',
    message: errorMessage,
  };
  // ...
  const source = m.cold('          --a---|', values, error);
  // ...
  const expectedUncaught = m.cold('--#', values, error);
  // ...
  const actualUncaught = source.pipe(
    map(() => {
      throw new Error(errorMessage);
    })
  );
  // ...
Note that to test the retry operator, we created a special Observable (getRetryTestObservable) that will error for the first n subscriptions before behaving normally afterward.
/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/6f944998/src/testUtils.ts
 */
// ...
export const getRetryTestObservable = <Error, Success>(
    errors: Error[],
    success: Success,
  ) =>
    () => {

  const errorsCopy = [...errors];
  const customObservable = new Observable((observer) => {
    const error = errorsCopy.shift();
    if (error !== undefined) {
      observer.error(error);
    } else {
      observer.next(success);
      observer.complete();
    }
  });
  return customObservable;
};
And here’s how we use it:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/6f944998/src/__tests__/basic.js
 */
// ...
  const errorMessage = 'This is an error';
  // ...
  const values = {
    x: successMessage,
  };
  const source = m.cold('   -a-|', values);
  const expected1 = m.cold('-x-|', values);
  // ...
  // This observable will error on the first 2 subscriptions
  // but succeed on the third subscription onward.
  const failTwiceThenSucceed = getRetryTestObservable(
    [errorMessage, errorMessage], successMessage
  )();

  const actual1 = source.pipe(
    switchMap(() => failTwiceThenSucceed.pipe(retry(2))),
  );
  m.expect(actual1).toBeObservable(expected1);
  // ...
Redux store and reducer
Let’s talk briefly about the redux store, reducer and actions used in this tutorial. They are all pretty simple. The store is split into two substores: url holding the URL input from the user and result holding the fetch result from GitHub, which can be either “initial” (blank user input), “success” with fetched file content, “failure” with error message or “fetch in progress”.

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/2c11ace1/src/types.ts
 */
// ...
export interface RootState {
  url: string;
  result: FetchResult;
}

export enum FetchStatus {
  Initial = 'Initial',
  InProgress = 'InProgress',
  Failed = 'Failed',
  Success = 'Success',
}

export type FetchResult = {
  status: FetchStatus.Initial,
} | {
  status: FetchStatus.InProgress,
} | {
  status: FetchStatus.Failed,
  message: string;
} | {
  status: FetchStatus.Success,
  data: string;
};
// ...
The actions are Flux Standard Actions that correspond to either URL updates or fetch status updates (success, fail, start etc).

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/2c11ace1/src/types.ts
 */
// ...
export enum ActionTypes {
  FETCH_BEGIN = 'FETCH_BEGIN',
  FETCH_SUCCESS = 'FETCH_SUCCESS',
  FETCH_FAIL = 'FETCH_FAIL',
  FETCH_INITIAL = 'FETCH_INITIAL',
  UPDATE_URL = 'UPDATE_URL',
}

export interface FetchBeginAction {
  type: ActionTypes.FETCH_BEGIN;
}

export interface FetchSuccessAction {
  type: ActionTypes.FETCH_SUCCESS;
  payload: {
    data: string;
  };
}

export interface FetchFailAction {
  type: ActionTypes.FETCH_FAIL;
  payload: {
    message: string;
  };
}

export interface FetchInitialAction {
  type: ActionTypes.FETCH_INITIAL;
}

export interface UpdateURLAction {
  type: ActionTypes.UPDATE_URL;
  payload: {
    url: string;
  };
}

export type Action =
  FetchBeginAction | FetchSuccessAction | FetchFailAction | FetchInitialAction |
  UpdateURLAction;
I won’t show the reducer here because it’s very simple and not very essential to this tutorial but you can see it here .

Creating user input epic
Now we have all the ingredients to create and test a redux-observable epic. Starting in version 1 of redux-observable, each epic is passed two parameters:

action$: an Observable representing a stream of dispatched actions after these actions have been processed by the reducer.
state$: an Observable representing a stream of Redux state changes.
From these, each epic has to return a new Observable representing a stream of actions to be dispatched to the store. Because our epic will manage data fetching from GitHub and we only want to fetch data when the URL changes, the first thing we want our epic to do is to only respond to UpdateURLActions by using the filter operator:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/2c11ace1/src/epic.ts
 */
// ...
const isUpdateURLAction = (action: Action): action is UpdateURLAction =>
  action.type === ActionTypes.UPDATE_URL;

export const epic =
  (action$: Observable<Action>) =>
    action$.pipe(
      filter<Action, UpdateURLAction>(isUpdateURLAction),
    );
In case you’re wondering, the strange-looking type annotation action is UpdateURLAction for isUpdateURLAction is a user-defined type guard, which is needed to satisfy the type checker due to the type definition of filter in RxJS.

Below is the marble test for the epic we’ve just written. We can see that the epic has filtered out action b because it is not an UpdateURLAction:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/2c11ace1/src/__tests__/epic.js
 */
// ...
test('Should only act on UpdateURLAction', marbles(m => {
  const values = {
    a: {type: ActionTypes.UPDATE_URL},
    b: {type: 'Unknown'},
  };
  const action$ = m.cold('  -a-b-a-aaa----------|', values);
  const expected$ = m.cold('-a---a-aaa----------|', values);

  const actual$ = epic(action$);

  m.expect(actual$).toBeObservable(expected$);
}));
// ...
The repo is now at commit 2c11ace1.

Next, because the URL updates originate from user input, we should debounce them with the debounceTime operator so that we don’t wastefully send out AJAX requests for every keystroke. Because timing works differently in marble testing than in production as shown above, we should allow the time duration and scheduler to be injectable so that the operator behaves normally during production but is completely controlled by the TestScheduler during testing. An easy way to do dependency injection is to replace epic with a function getEpic that receives parameters to customize the epic for testing but uses default values when in production:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/59546414/src/epic.ts
 */
// ...
export const getEpic = (
    // These arguments allow for dependency injection during testing:
    dueTime: number = 250,
    // Note: `undefined` scheduler passed to `debounceTime` in production means
    // it'll use the "natural" scheduler:
    scheduler: Scheduler | undefined = undefined,
  ) =>
    (action$: Observable<Action>) =>
      action$.pipe(
        filter<Action, UpdateURLAction>(isUpdateURLAction),
        debounceTime(dueTime, scheduler),
      );
Below is our updated marble test. Note that the five UpdateURLActions in the last version have been debounced into only one in this version:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/59546414/src/__tests__/epic.js
 */
import {
  getEpic,
} from '../epic';
// ...
test('Should only act on UpdateURLAction', marbles(m => {
  const values = {
    a: {type: ActionTypes.UPDATE_URL},
    b: {type: 'Unknown'},
  };
  const action$ = m.cold('  -a-b-a-aaa----------|', values);
  const expected$ = m.cold('-------------a------|', values);

  // Inject `dueTime` and `TestScheduler` into epic:
  const epic = getEpic(4, m.scheduler);

  const actual$ = epic(action$);

  m.expect(actual$).toBeObservable(expected$);
}));
// ...
One immediate advantage of dependency injection is that we can choose a very small debounce duration of four “frames” for ease of testing. The repo is now at commit 59546414.

Access redux state from state$ stream
In order to make a fetch request, we need to get the GitHub URL from which to fetch a file’s content. We could easily have gotten it from the payload of UpdateURLActions but for demonstration purposes, we’ll use the new state$ stream provided by redux-observable:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/cf389ab0/src/epic.ts
 */
// ...
      action$.pipe(
        filter<Action, UpdateURLAction>(isUpdateURLAction),
        debounceTime(dueTime, scheduler),
        withLatestFrom(state$),
      );
The withLatestFrom operator combines the current stream’s value with the most recent value from another stream. Below is the marble diagram showing how withLatestFrom works in isolation to jog your memory. Note how a from the source stream is combined with s from the other stream into the array ['a', 's']:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/cf389ab0/src/__tests__/basic.js
 */
// ...
  const values = {
    x: ['a', 's'],
    y: ['b', 't'],
    z: ['c', 't'],
  };
  const source = m.cold('  -a---b---c--|');
  const other = m.cold('   s--t--------|');
  const expected = m.cold('-x---y---z--|', values);
  const actual = source.pipe(
    withLatestFrom(other),
  );
  // ...
Now that our epic uses state$, we also have to create a state$ stream in our marble test, which, for the purpose of testing, only emits a single value. Not that state$ starts before the action$ stream to replicate how redux-observable actually works:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/cf389ab0/src/__tests__/epic.js
 */
// ...
  const stateValue = {url: 'http://example.com'};
  const urlUpdateAction = {type: ActionTypes.UPDATE_URL};

  const values = {
    a: urlUpdateAction,
    b: {type: 'Unknown'},
    s: stateValue,
    x: [urlUpdateAction, stateValue],
  };

  const state$ = m.cold('  s----------------------', values);
  const action$ = m.cold('  -a-b-a-aaa----------|', values);
  const expected$ = m.cold('-------------x------|', values);

  // Inject `dueTime` and `TestScheduler` into epic:
  const epic = getEpic(4, m.scheduler);

  const actual$ = epic(action$, state$);

// ...
The repo is now at commit cf389ab0.

Add data fetching
A few quick notes before we dive into data fetching:

When you browse a file in GitHub’s web interface, say https://github.com/ReactiveX/rxjs/blob/master/index.js, the URL to which to send a GET request to get that file’s content is slightly different: https://api.github.com/repos/ReactiveX/rxjs/contents/index.js?ref=master. If the URL points to a file, the AJAX response from GitHub is a JSON object, of which we only care about the content property, a base64-encoded string of the file’s content:
{
  "name": "index.js",
  "path": "index.js",
  "sha": "6c3b0dc17947969cf907e94dac9426b6effcc53f",
  "size": 47,
  "url": "https://api.github.com/repos/ReactiveX/rxjs/contents/index.js?ref=master",
  "html_url": "https://github.com/ReactiveX/rxjs/blob/master/index.js",
  "git_url": "https://api.github.com/repos/ReactiveX/rxjs/git/blobs/6c3b0dc17947969cf907e94dac9426b6effcc53f",
  "download_url": "https://raw.githubusercontent.com/ReactiveX/rxjs/master/index.js",
  "type": "file",
  "content": "bW9kdWxlLmV4cG9ydHMgPSByZXF1aXJlKCcuL2Rpc3QvcGFja2FnZS9SeCcp\nOwo=\n",
  "encoding": "base64",
  "_links": {
    "self": "https://api.github.com/repos/ReactiveX/rxjs/contents/index.js?ref=master",
    "git": "https://api.github.com/repos/ReactiveX/rxjs/git/blobs/6c3b0dc17947969cf907e94dac9426b6effcc53f",
    "html": "https://github.com/ReactiveX/rxjs/blob/master/index.js"
  }
}
However, if the URL points to a directory, the response will be an array in which each element looks like the JSON object above.

To make an AJAX request in RxJS, we use the ajax method to get an Observable that will resolves to either a response or an error. However, in testing, we need the ability to inject a custom Observable in order to control the AJAX request’s behavior e.g. make it succeed, fail or fail after n tries then succeed. As such, the epic accepts as parameter a function getAjax that defaults to RxJS’s ajax method in production but can return a custom Observable in testing.

We should emit a “fetch begins” action before sending out the AJAX request and a “fetch fail” with the appropriate message when it fails.

Based on the above pointers, we can now fetch data from GitHub by mapping each debounced and filtered UpdateURLAction to an AJAX Observable. Note that because mapping each URL to an AJAX Observable creates an Observable of Observable, we need to flatten the result by using switchMap instead of just map.

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/722e39bb/src/epic.ts
 */
// ...
export const getEpic = (
    // These arguments allow for dependency injection during testing:
    getAjax: typeof ajax = ajax,
    dueTime: number = 250,
    // Note: `undefined` scheduler passed to `debounceTime` in production means
    // it'll use the "natural" scheduler:
    scheduler: Scheduler | undefined = undefined,
  ) =>
    (action$: Observable<Action>, state$: Observable<RootState>) =>
      action$.pipe(
      // ...
        switchMap<[UpdateURLAction, RootState], FetchOutcome>(([, {url}]): Observable<FetchOutcome> => {
          if (url === '') {
            // If the user input is blank, show the initial prompt:
            return of<FetchInitialAction>({type: ActionTypes.FETCH_INITIAL});
          } else {
            const parseResult = parse(url);
            if (parseResult === null) {
              // If we cannot parse the URL into a GitHub URL at all, show an error message:
              return of<FetchFailAction>({type: ActionTypes.FETCH_FAIL, payload: {message: 'Invalid GitHub URL'}});
            } else {
              // Otherwise, initiate a fetch:
              const fetchBegin$ = of<FetchBeginAction>({type: ActionTypes.FETCH_BEGIN});
              const {branch, path, repo, user} = parseResult;
              const fetchURL = `https://api.github.com/repos/${user}/${repo}/contents/${path}?ref=${branch}`;

              let ajaxRequest: AjaxRequest;
              if (GITHUB_TOKEN === undefined) {
                // If no github token is provided, send an unauthenticated request:
                ajaxRequest = {
                  url: fetchURL,
                };
              } else {
                // Otherwise, send credentials along to avoid rate limit:
                ajaxRequest = {
                  url: fetchURL,
                  headers: {
                    Authorization: `token ${GITHUB_TOKEN}`,
                  },
                };
              }

              const fetchPipeline$ = getAjax(ajaxRequest).pipe(
                map<AjaxResponse, FetchSuccessAction>(({response}) => {
                  if (Array.isArray(response)) {
                    // This means the fetched URL is a directory instead
                    // of a file:
                    throw new Error('URL is a GitHub directory instead of a file');
                  }
                  const {content} = response;
                  const decoded = atob(content);
                  return {type: ActionTypes.FETCH_SUCCESS, payload: {data: decoded}};
                }),

              );
              return concat(fetchBegin$, fetchPipeline$);
            }
          }
        }),
        // ...
In the marble test, we can now inject a custom Observable testAjax that succeeds with some dummy data after waiting for one frame (to simulate real-world conditions). We can see that the correct “fetch success” action is emitted by the epic:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/722e39bb/src/__tests__/epic.js
 */
// ...
  const stateValue = {url: 'https://github.com/ReactiveX/rxjs/blob/master/index.js'};
  const urlUpdateAction = {type: ActionTypes.UPDATE_URL};

  const contentAsString = 'This is a test.';
  // Encode to base64 for testing:
  const contentAsBase64 = btoa(contentAsString);

  const values = {
    a: urlUpdateAction,
    b: {type: 'Unknown'},
    s: stateValue,
    x: {type: ActionTypes.FETCH_BEGIN},
    y: {type: ActionTypes.FETCH_SUCCESS, payload: {data: contentAsString}},
  };

  const state$ = m.cold('  s----------------------', values);
  const action$ = m.cold('  -a-b-a-aaa----------|', values);
  const expected$ = m.cold('-------------xy-----|', values);


  const testAjax = jest.fn().mockReturnValueOnce(
    of({response: { content: contentAsBase64}}).pipe(delay(1))
  );

  // Inject `dueTime` and `TestScheduler` into epic:
  const epic = getEpic(testAjax, 4, m.scheduler);
  // ...
The repo is now at commit 722e39bb.

Handling errors
So far, we’ve only handled one type of error: the user input is not a valid GitHub URL at all. Other errors can happen. An AJAX request can fail because of a network error, because the resource that the URL points to doesn’t exist on GitHub (404s) or because the URL points to a directory instead of a file. We can provide an error handler by retrying failed requests three times and adding the catchError operator within the fetchPipeline$ stream:

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/320ed09b/src/epic.ts
 */
// ...
        switchMap<[UpdateURLAction, RootState], FetchOutcome>(([, {url}]): Observable<FetchOutcome> => {
        // ...
              const fetchPipeline$ = getAjax(ajaxRequest).pipe(
                retry<AjaxResponse>(3),
                map<AjaxResponse, FetchSuccessAction>(({response}) => {
                // ...
                }),
                catchError<any, FetchFailAction>((err: AjaxError) => {
                  // Try to use error message in ajax response because we assume
                  // it's more relevant that the `message` property of the `err`
                  // observable:
                  let message: string;
                  if (err.xhr && err.xhr.response && err.xhr.response.message) {
                    message = err.xhr.response.message;
                  } else {
                    message = err.message;
                  }
                  return of<FetchFailAction>({type: ActionTypes.FETCH_FAIL, payload: {message}});
                }),
              );
              // ...
        }),
        // ...
Note that the location of the catchError operator is important. We put it within switchMap but not in action$’s pipeline (action$.pipe(...)) in order to prevent any error from bubbling up to action$’s pipeline. If that is allowed to happen, whatever Observable the catchError returns will replace the entire epic, causing the epic to terminate and stop listening to new actions.

To test that the error handling actually works, we need to introduce some AJAX errors by making the special getRetryTestObservable we created earlier fail a certain number of times. By making it fail three times, we test that AJAX requests are indeed retryed three times. By making it fail four times, we test that the epic emits a “fetch fail” action when it has exhausted all the retrys.

/**
 * https://github.com/huy-nguyen/redux-observable-marble-diagrams/blob/320ed09b/src/__tests__/epic.js
 */
// ...
it('Should retry when encountering fetch error', marbles(m => {
  const contentAsString = 'This is a test.';
  const contentAsBase64 = btoa(contentAsString);

  const ajaxSuccess = new AjaxResponse(undefined, {
    status: 200, response: {content: contentAsBase64}, responseType: 'json',
  });
  const ajaxFailure = new AjaxError('This is an error', {
    status: 404, response: {message: 'Not Found'}, responseType: 'json',
  });

  // The AJAX request will fail three times and succeed when retried on the fourth time:
  const getTestAjaxObservable = getRetryTestObservable([ajaxFailure, ajaxFailure, ajaxFailure], ajaxSuccess);

  const values = {
    a: {type: ActionTypes.UPDATE_URL},
    b: {type: 'Unknown'},
    s: {url: 'https://github.com/ReactiveX/rxjs/blob/master/index.js'},
    x: {type: ActionTypes.FETCH_BEGIN},
    y: {type: ActionTypes.FETCH_SUCCESS, payload: {data: contentAsString}},
  };

  const state$ = m.cold(' s-------------------------', values);
  const action$ = m.cold('  -a-b-a-aaa----------|', values);
  const expected$ = m.cold('-------------(xy)---|', values);
  // ...
}));
// ...
it('Should dispatch "fetch fail" action when retries are unsuccessful due to 404s', marbles(m => {
  const errorMessage = 'Not Found';

  const ajaxFailure = new AjaxError('This is a test error', {
    status: 404, response: {message: errorMessage}, responseType: 'json',
  });

  // The AJAX request will fail four times and succeed when retried on the fifth time.
  // However, because the epic only retries three times, only errors will be emitted:
  const getTestAjaxObservable = getRetryTestObservable(
    [ajaxFailure, ajaxFailure, ajaxFailure, ajaxFailure], undefined
  );
  const values = {
    a: {type: ActionTypes.UPDATE_URL},
    b: {type: 'Unknown'},
    s: {url: 'https://github.com/ReactiveX/rxjs/blob/master/index.js'},
    x: {type: ActionTypes.FETCH_BEGIN},
    z: {type: ActionTypes.FETCH_FAIL, payload: {message: errorMessage}},
  };

  const state$ = m.cold(' s-------------------------', values);
  const action$ = m.cold('  -a-b-a-aaa----------|', values);
  const expected$ = m.cold('-------------(xz)---|', values);
  // ...
}));
// ...
We’re now done with creating the pic. The repo is now at commit 320ed09b.
