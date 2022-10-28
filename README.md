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

![Alt text](/assets/of-screen.png "Optional title")
