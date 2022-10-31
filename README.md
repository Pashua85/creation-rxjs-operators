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

`scheduler` - это планировщик. Планировщики не являются темой этой статьи, но если кратко, то это объекты, с помощью которых можно влиять на
время и порядок выполнения колбеков подписки и колбеков из операторов в `pipe`, а конкретнее - можно задать, как именно эти функции попадут в стэк вызовов: напрямую, через очереди микротасков, макротасков или очередь браузера для отрисовки содержимого. Подробнее можно почитать [тут](https://habr.com/ru/post/529000/) и посмотреть [этот доклад](https://www.youtube.com/watch?v=S1eDh7MonbI&t=131s).

Но вернемся к оператору `from`. Я не случайно использовал словосочетание "итерируемое значение", а не "итерируемый объект", потому что `from` работает еще и с примитивом `string`, сообщая в поток знаки из строки по-одному.

```ts
from(new Set([1, 2, 2, 3, 3])
  .subscribe({
	next: value => console.log(“next:”, value),
  })
// next: 1
// next: 2
// next: 3

from(“String”)
  .subscribe({
	next: value => console.log(“next:”, value),
  })
// next: S
// next: t
// next: r
// next: i
// next: n
// next: g

from(new Promise((resolve) => resolve('I am a promise!'))).subscribe({
  next: (value) => console.log('next: ', value),
});
// next: I am a promise!
```

А [вот вариант](https://stackblitz.com/edit/rxjs-rfc9rd) приложения с хорошими мальчиками, в котором используется `from` для того, чтобы в поток попали не все url-ы картинок сразу, а по одной (допустим для того, чтобы они появлялись по очереди c полсекунды). Правда при этом пришлось отдельно позаботиться об обработке ошибок, потому что и адрес картинки, и сообщение об ошибке - это строка и просто проверка на тип уже не работает. Также тут `from` используется, чтобы в потоке работать уже с результатом обещания из fetch-запроса:

```ts
fromEvent(
  ...
)
  .pipe(
    switchMap((subBreed) => {
      ...

      return from(fetch(url).then((res) => res.json())).pipe(
        switchMap((res) => {
          if (res.status === 'error') {
            return throwError(() => new Error());
          }
          return from(res.message);
        }),
        concatMap((val) => {
          return of(val).pipe(delay(500));
        })
      );
    }),
    catchError(() => {
      return of('Error from catch error');
    })
  )
  .subscribe((val) => {
    if (val === 'Error from catch error') {
    ...
    }
    ...
  });
```

## interval

Перейдем к работе со временем. Оператор `interval` создает Observable, который сообщает в поток целые числа, начиная от 0, в поток с указанной периодичностью (в милисекундах).

```ts
interval(period: number = 0, scheduler: SchedulerLike = asyncScheduler): Observable<number>
```

![interval](/assets/interval.png)

```ts
interval(1000).pipe(take(4));
```

![interval](/assets/interval.gif)

Перейдём к конкретному [примеру](https://stackblitz.com/edit/rxjs-jmp1qy) в песочнице. Тут запуск интервала начинает смену цвета дива на рандомный через каждую секунду.

```ts
startIntervalBtn.addEventListener('click', () => {
  ...
  intervalSub = interval(1000)
    ...
    .subscribe((val) => {
      ...
      intervalNumber.style.backgroundColor = getRandomColor();
    });
});
```

Как вы уже наверное заметили, первое значение сообщается в поток через 1 секунду. А что если нам нужно сначала выполнить колбек подписки сразу после клика, а затем его повторять с указанной периодичностью? Как вариант, можно сразу передать в поток значение оператором `startWith`. Вот измененный [пример](https://stackblitz.com/edit/rxjs-te6leg)

```ts
  intervalSub = interval(1000)
    .pipe(
      startWith(0),
      ...
    )
    .subscribe((val) => {
      intervalNumber.innerText = val.toString();
      ...
    });
```

Если бы в колбеке подписки не имело бы значение какое число попадает в поток, как в прошлом примере, то все было бы хорошо. Но тут добавлена логика, завязанная на значение из потока, a как мы видим,
в поток дважды приходит ноль (один раз его эмитит оператор `startWith`, второй - Observable, который создан с помощью `interval` ).
Можно было бы изначально начать с -1 (`startWith(-1)`), а в функции подписки делать поправку на единицу, но к счастью, есть другой оператор, с помощью которого можно реализовать все проще - `timer`.
