---
title: 'Concurrent task queue using RxJS'
date: 2022-05-15 00:00:00 +0200
categories: [Frontend, RxJS]
tags: [javascript, rxjs, queue, concurrency, task, observable, observer, subject]
---

## So far so good
---
Web applications have always needed to run different tasks based on events triggered by the end-user. Long ago, when browsers were more restricted, these applications had to run tasks one by one in most cases.

Currently, browsers have more capabilities which allow more complex and demanding scenarios, such as the execution of concurrent tasks from a web application.

With the rise of **SPAs** (Single-Page Applications), some Javascript libraries or frameworks favor the use of [**Promises**](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise){:target="_blank"} while others favor [**Observables**](https://rxjs.dev/guide/observable){:target="_blank"}. With either of them, it is possible to successfully manage the execution of tasks concurrently, either by using [**Promise.all**](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all){:target="_blank"} in the case of **Promises** or by using combination operators such as [**forkJoin**](https://rxjs.dev/api/index/function/forkJoin){:target="_blank"} in the case of **Observables**. All of these things make life easier, although it is not enough for some scenarios.

Let's imagine a SPA to store and tag videos where the end-user can upload videos while navigating through the different screens performing actions that request and load data. This specific case will be somewhat troublesome since all requests are [**XHR**](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest){:target="_blank"} and, as is known, current browsers still have limitations in this regard.

If the end-user of this imaginary application needs to upload many videos in the background while still browsing it, we need some way to limit the number request running in the background to free up some connections to continue browsing, otherwise requests will start to be blocked once the limit set by the browser is exceeded. If you want to learn more about browser restrictions, click [here](https://www.linkedin.com/pulse/why-does-your-browser-limit-number-concurrent-ishwar-rimal/){:target="_blank"}.

**RxJS** to the rescue! 🦸

## How the hell do we do this?
---

### Context

Since we are talking about uploading videos, we are going to represent the upload process by means of an entity called **process** that has a unique `id`, a `status` (`pending`, `in progress`, `completed` and `failed`), a `percentage` of progress (from `0` to `100`) and the metadata and content of the video in a `data` property.

```javascript
{
  id: 1,
  status: 'pending',
  percentage: 0,
  data: {...}
}
```

### Concepts

Before we get down to business, it's important to know what a [**Subject**](https://rxjs.dev/guide/subject){:target="_blank"} is in the RxJS world.

In a nutshell, a **Subject** is a special type of **Observable** that allows multicasting unlike plain Observables that allow unicasting, but it also acts as an **Observer** allowing values to be sent through the **Subject** itself.

This piece of **RxJS** guides the rest of the lines in this post.

### Code

Let's to discover the code step by step for a better understanding.

First, we need a **Subject** that acts as the source where we are going to feed the new processes that have to be executed. For this purpose there is `addProcess$` which receives raw processes represented by the `id` and `data` properties mentioned above.

```javascript
const addProcess$ = new Subject();
```

Every time a new raw proccess is fed into the Subject, it goes trough a number of different status depending on the state of the concurrent queue that we are trying to build:

1. A new process comes in: the process remains with `status` equal `pending` and `percentage` equal to `0`.
2. A pending process is scheduled for execution: the process changes the `status` to `in progress` and `percentage` equal to the progress of the task, in this case, the HTTP request.
3. A running process completes or fails: the process changes the `status` to `completed` or `failed` depending on how it ends.

Keeping this in mind, we need a way to map the raw model to a meaninful model that allow us to represent that world.

```javascript
const addProcess$ = new Subject();
const processQueue$ = addProcess$
  .pipe(
    connect(subject$ =>
      subject$.pipe(
        mergeMap((process) => {
          return of(updateProcess(process, 'pending', 0));
        })
      )
    )
  ).subscribe(process => { ... })

const updateProcess = (process, status, percentage) => ({
  ...process,
  status,
  percentage,
});
```

The **connect** operator allows multicasting of the source, so it shares the single subscription created with other subscribers, which is efficient in terms of subscription management and, on the other hand, all the subscribers that may exist would receive the same data.

It's also necessary the use of **mergeMap** operator to project the value of the source to end up mapping to a model that represents the state mentioned above in the first point.

>
- [x] A new process comes in: the process remains with `status` equal `pending` and `percentage` equal to `0`.
- [ ] A pending process is scheduled for execution: the process changes the `status` to `in progress` and `percentage` equal to the progress of the task, in this case, the HTTP request.
- [ ] A running process completes or fails: the process changes the `status` to `completed` or `failed` depending on how it ends.

To address the problem of scheduling the execution processes, let's take a look at the following code snippet 🤓:

```javascript
const maxConcurrency = 2;
const addProcess$ = new Subject();
const processQueue$ = addProcess$
  .pipe(
    connect(subject$ =>
      merge(
        subject$.pipe(
          mergeMap((process) => {
            return of(updateProcess(process, 'pending', 0))
              .pipe(
                observeOn(asyncScheduler),
                takeUntil(merge(subject$))
              );
          })
        ),
        subject$.pipe(
          mergeMap((process) => {
            return fakeHttpRequest(process)
              .pipe(
                startWith(updateProcess(process, 'in progress', 0)),
                endWith(updateProcess(process, 'completed', 100)),
                catchError((error) => of(updateProcess(process, 'failed', process.percentage))),
                scan((accum, value) => ({ ...accum, ...value }), {}),
              )
          }, maxConcurrency)
        )
      ).pipe(
        scan((accum, value) => ({ ...accum, [value.id]: value }), {}),
        map(obj => Object.values(obj))
      )
    )
  ).subscribe(processes => { ... })

const updateProcess = (process, status, percentage) => ({
  ...process,
  status,
  percentage,
});

const fakeHttpDelay = 5000;
const fakeHttpRequest = (process) => new Observable(subscriber => {
  subscriber.next(updateProcess(process, 'in progress', 25));
  setTimeout(() => {
    subscriber.next(updateProcess(process, 'in progress', 50));

    if (Math.floor(Math.random() * 10) % 2 === 0)
      subscriber.error(new Error('Connection error!'));

    setTimeout(() => {
      subscriber.next(updateProcess(process, 'in progress', 75));
      setTimeout(() => {
        subscriber.next(updateProcess(process, 'in progress', 99));
        subscriber.complete();
      }, fakeHttpDelay);
    }, fakeHttpDelay);
  }, fakeHttpDelay);
});
```

The first new thing we see is that just below the **connect** operator, there is a **merge** operator that allows us to declare another **Observable** to manage the scheduling of process executions.

The second **Observable** also uses a **mergeMap** operator to project the value of the source and subsequently fires an HTTP request (*fakeHttpRequest*) that simulates the process of uploading a video. Note that **mergeMap** receives the `maxConcurrency` variable as its second parameter, which establishes the maximum number of input **Observables** being subscribed to concurrently. This value sets how many upload processes are executed concurrently, leaving free connections to continue browsing.

As for the operators within the `pipe` block:

* **startWith** sets the state at the moment of subscription.
* **endWith** sets the state immediately after the source completes.
* **catchError** sets the state when an error occurs.
* **scan** allows managing the state by acummulating or merging intermediate states.

Before finishing, it's worth mentioning the pair of statements added to the first **Observable** inside the operator **merge**:
* **observeOn(asyncScheduler)** defer the emission of the data from the **Observable** to the JS event loop for better performance.
* **takeUntil(merge(subject$))** unsubscribe the **Observable** once it emits a value. The internal **merge** is a trick to make it emit once, since `subject$` has already emitted a value when it got here.

The above describes the states previously mentioned in the second and third points.

>
- [x] A new process comes in: the process remains with `status` equal `pending` and `percentage` equal to `0`.
- [x] A pending process is scheduled for execution: the process changes the `status` to `in progress` and `percentage` equal to the progress of the task, in this case, the HTTP request.
- [x] A running process completes or fails: the process changes the `status` to `completed` or `failed` depending on how it ends.

### Bonus code

What if you want to drop some scheduled process on the queue? There goes a solution based on introducing a new Subject where the process to be discarded is notified. ✨

In a few words, merging what we already had with that new Subject and using a scan operator, we manage to discard those processes that are in `processQueue$` when they come through `dropProcess$`.

```javascript
const maxConcurrency = 2;
const addProcess$ = new Subject();
const dropProcess$ = new Subject();
const processQueue$ = addProcess$
  .pipe(
    connect(subject$ =>
      merge(
        subject$.pipe(
          mergeMap((process) => {
            return of(updateProcess(process, 'pending', 0))
              .pipe(
                observeOn(asyncScheduler),
                takeUntil(merge(subject$))
              );
          })
        ),
        subject$.pipe(
          mergeMap((process) => {
            return fakeHttpRequest(process)
              .pipe(
                startWith(updateProcess(process, 'in progress', 0)),
                endWith(updateProcess(process, 'completed', 100)),
                catchError((error) => of(updateProcess(process, 'failed', process.percentage))),
                scan((accum, value) => ({ ...accum, ...value }), {}),
                takeUntil(
                  dropProcess$.pipe(
                    filter(p => p.id === process.id)
                  )),
              )
          }, maxConcurrency)
        )
      )
    )
  );

const droppableProcessQueue$ = merge(processQueue$, dropProcess$)
  .pipe(
    scan((accum, value) => {
      if (value.status === 'dropped') {
        const { [value.id]: _, ...rest } = accum;
        return rest;
      }
      return { ...accum, [value.id]: value };
    }, {}),
    map(obj => Object.values(obj))
  );

droppableProcessQueue$.subscribe(processes => {...});

const updateProcess = (process, status, percentage) => ({
  ...process,
  status,
  percentage,
});

const fakeHttpDelay = 5000;
const fakeHttpRequest = (process) => new Observable(subscriber => {
  subscriber.next(updateProcess(process, 'in progress', 25));
  setTimeout(() => {
    subscriber.next(updateProcess(process, 'in progress', 50));

    if (Math.floor(Math.random() * 10) % 2 === 0)
      subscriber.error(new Error('Connection error!'));

    setTimeout(() => {
      subscriber.next(updateProcess(process, 'in progress', 75));
      setTimeout(() => {
        subscriber.next(updateProcess(process, 'in progress', 99));
        subscriber.complete();
      }, fakeHttpDelay);
    }, fakeHttpDelay);
  }, fakeHttpDelay);
});
```

### Sample application

Below is a screenshot of an application that implements all the code described in the post for testing purposes.

![app](/assets/img/posts/concurrent-task-queue-using-rxjs/sample-app.jpg)
_Sample application_

The full source code of the sample application is [**here**](https://github.com/jcarloslr10/concurrent-task-queue-using-rxjs){:target="_blank"}.

## Conclusions
---

To wrap this up:

- The power of the RxJS library and all its components and operators is evident, it really provides a range of fascinating options.

- It becomes clear that in many scenarios RxJS is as powerful as it's complex. Sometimes, a deep knowledge is needed when implementing this type of solutions.

I hope you like it and it clarifies a lot of the black magic that RxJS has.

See you! 😉
