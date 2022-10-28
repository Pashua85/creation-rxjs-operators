# Операторы создания в rxjs

В этой статье рассматриваются некоторые из наиболее часто используемых операторов для создания Observable.

## of

Этот оператор создает Observable просто на основе значения, которое мы в него передаем. Если принимает Promise, то в поток,
для которого источником является созданный Observable, попадет непосредственно сам промис, если принимает массив - то попадет массив (аналогично с другими итерируемыми объектами). Оператор может принимать несколько аргументов и все они последовательно попадут в поток такими, как есть:

```ts
function someFunction(): void {}

of(
  new Set([1, 2, 2, 3, 3]),
  someFunction,
  "some string",
  new Promise((resolve) => resolve("I am a promise!"))
).subscribe({
  next: (value) => console.log(`next (of): ${value}`),
});

// from console:
// next (of): [object Set]
// next (of): function someFunction() { }
// next (of): some string
// next (of): [object Promise]
```

Сигнатура у этого оператора следующая:

```ts
of(value: T): Observable<T>
```

Пример с использованием "of" можно посмотреть вот в [этой песочнице](https://stackblitz.com/edit/rxjs-rghhyk):

![Of example scneenshot](/assets/of-screen.png)

По нажатию на соответствующую кнопку, подгружаются урлы картинок собак определенной породы. В `data.message` приходит либо массив с адресами картинок, либо текст ошибки:

```ts
fromEvent(
  buttons,
  'click',
  (e: HTMLElementEvent<HTMLButtonElement>) => e.target.dataset.key
)
  .pipe(
    switchMap((subBreed) => {
      ...
      const url = `https://dog.ceo/api/breed/spaniel/${subBreed}/images/random/4`;

      return from(fetch(url).then((res) => res.json())).pipe(
        map((data: Data) => {
          return data.message;
        }),
        ...
      );
    }),
  )
  .subscribe((val) => {
    imagesContainer.innerHTML = '';
    if (typeof val === 'string') {
      ...
    }

    if (Array.isArray(val)) {
      ...
    }
  });
```

При этом для того, чтобы не отправлять запрос для одной и той же породы повторно, можно закэшировать ответ. И если картинки для данной
породы уже имеются, то `switchMap` поменяет поток с событиями клика не на поток с результатами запроса, а на поток из Observable, созданный
c помощью оператора `of`. Это позволяется ничего не менять в колбэке подписки:

```ts
fromEvent(
  ...
)
  .pipe(
    switchMap((subBreed) => {
      if (cashed.has(subBreed)) {
        return of(cashed.get(subBreed));
      }
      ...

      return from(fetch(url).then((res) => res.json())).pipe(
        ...
        tap((data) => {
          cashed.set(subBreed, data);
        })
      );
    }),
  )
  .subscribe((val) => {
    ...
  });
```

## from

Оператор `from` создает Observable только на основе итерируемого значения или промиса. В случае обещания в поток попадет результат его выполнения, а при работе с итерируемым значением элементы этого значения попадают в поток по очереди.

```ts
from<T>(input: ObservableInput<T>, scheduler?: SchedulerLike): Observable<T>
```

![from](/assets/from.png)
