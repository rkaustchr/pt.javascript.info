# Erros personalizados, estendendo Error

Quando desenvolvemos algo, precisamos freqüentemente das nossas próprias classes de erro para refletir coisas específicas que podem dar errado na nossa tarefa. Para erros em operações de rede poderíamos precisar de um `HttpError`, em operações de banco de dados um `DbError`, em operações de busca um `NotFoundError`, etc.

Nossos erros deveriam suportar propriedades básicas de erro como `message`, `name` e, preferencialmente, `stack`. Mas eles também podem ter outras propriedades próprias, por exemplo, objetos `HttpError` podem ter a propriedade `statusCode` com valores como `404` ou `403` ou `500`.

JavaScript nos permite usar `throw` com qualquer argumento, então tecnicamente nossa classe de erro personalizada não precisa herdar de `Error`. Mas se nós herdarmos, então torna-se possível usar `obj instanceof Error` para identificar objetos de erro. Logo é melhor herdar.

Com o crescimento da aplicação, nossos próprios erros formam naturalmente uma hierarquia. Por exemplo, `HttpTimeoutError` pode herdar de `HttpError`, etc.

## Estendendo Error

Como exemplo, vamos considerar a função `readUser(json)` que deve ler um JSON com os dados do usuário.

Aqui segue um exemplo de como um `json` válido pode parecer:
```js
let json = `{ "name": "John", "age": 30 }`;
```

Internamente, vamos usar o `JSON.parse`. Se ele recebe um `json` malformado, ele lança um `SyntaxError`.

Mas mesmo que o `json` esteja sintaticamente correto, isso não significa que seja um usuário válido, certo? Pode estar faltando os dados obrigatórios. Por exemplo, pode não ter as propriedades `name` e `age` que são essenciais para nossos usuários.

Nossa função `readUser(json)` não vai apenas ler o JSON, mas também vai validar os dados. Se não houver os dados obrigatórios, ou o formato está errado, então é um erro. E não é um `SyntaxError`, porque os dados estão sintaticamente corretos, mas outro tipo de erro. Vamos chamar `ValidationError` e criar uma classe pra isso. Um erro desse tipo também tem que ter as informações sobre qual campo está errado.

Nossa classe `ValidationError` deve herdar a classe `Error`.

A classe `Error` é interna do JavaScript, mas aqui segue seu código aproximado para podermos entender o que estamos estendendo:


```js
// O "pseudocódigo" para a classe interna Error definida pelo próprio JavaScript
class Error {
  constructor(message) {
    this.message = message;
    this.name = "Error"; // (nomes diferentes para diferentes classes internas de erro)
    this.stack = <nested calls>; // não é padrão, mas a maioria dos ambientes suporta
  }
}
```

Agora vamos continuar e herdar `ValidationError` dela:

```js run untrusted
*!*
class ValidationError extends Error {
*/!*
  constructor(message) {
    super(message); // (1)
    this.name = "ValidationError"; // (2)
  }
}

function test() {
  throw new ValidationError("Whoops!");
}

try {
  test();
} catch(err) {
  alert(err.message); // Whoops!
  alert(err.name); // ValidationError
  alert(err.stack); // uma lista de chamadas aninhadas com número da linha de cada
}
```

Por favor, dê uma olhada no construtor:

1. Na linha `(1)` chamamos o construtor pai. JavaScript nos força chamar `super` no construtor filho, então isso é obrigatório. O construtor pai define a propriedade `message`.
2. O construtor pai também define a propriedade `name` como `"Error"`, então na linha `(2)` corrigimos para o valor certo.

Vamos tentar usar isso na `readUser(json)`:

```js run
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = "ValidationError";
  }
}

// Uso
function readUser(json) {
  let user = JSON.parse(json);

  if (!user.age) {
    throw new ValidationError("No field: age");
  }
  if (!user.name) {
    throw new ValidationError("No field: name");
  }

  return user;
}

// Exemplo funcional com try..catch

try {
  let user = readUser('{ "age": 25 }');
} catch (err) {
  if (err instanceof ValidationError) {
*!*
    alert("Invalid data: " + err.message); // Invalid data: No field: name
*/!*
  } else if (err instanceof SyntaxError) { // (*)
    alert("JSON Syntax Error: " + err.message);
  } else {
    throw err; // erro desconhecido, relance o erro (**)
  }
}
```

O bloco `try..catch` no código acima trata tanto o nosso erro  `ValidationError` quanto o erro interno `SyntaxError` do `JSON.parse`.

Observe como usamos o `instanceof` para verificar um tipo de erro específico na linha `(*)`.

Podemos também verificar em `err.name`, assim:

```js
// ...
// ao invés de (err instanceof SyntaxError)
} else if (err.name == "SyntaxError") { // (*)
// ...
```

A versão com `instanceof` é muito melhor, porque no futuro vamos estender `ValidationError`, fazer subtipos dela, como `PropertyRequiredError`. E a verificação com `instanceof` continuará funcionando para as novas classes herdeiras. Então isso é à prova de futuro.

Também é importante que caso o `catch` encontre um erro desconhecido, ele relance isso na linha `(**)`. O bloco `catch` só sabe tratar erros de validação e sintaxe, outros tipos de erro (causados por digitação do código ou outras razões desconhecidas) devem atravessar.

## Further inheritance

The `ValidationError` class is very generic. Many things may go wrong. The property may be absent or it may be in a wrong format (like a string value for `age` instead of a number). Let's make a more concrete class `PropertyRequiredError`, exactly for absent properties. It will carry additional information about the property that's missing.

```js run
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = "ValidationError";
  }
}

*!*
class PropertyRequiredError extends ValidationError {
  constructor(property) {
    super("No property: " + property);
    this.name = "PropertyRequiredError";
    this.property = property;
  }
}
*/!*

// Usage
function readUser(json) {
  let user = JSON.parse(json);

  if (!user.age) {
    throw new PropertyRequiredError("age");
  }
  if (!user.name) {
    throw new PropertyRequiredError("name");
  }

  return user;
}

// Working example with try..catch

try {
  let user = readUser('{ "age": 25 }');
} catch (err) {
  if (err instanceof ValidationError) {
*!*
    alert("Invalid data: " + err.message); // Invalid data: No property: name
    alert(err.name); // PropertyRequiredError
    alert(err.property); // name
*/!*
  } else if (err instanceof SyntaxError) {
    alert("JSON Syntax Error: " + err.message);
  } else {
    throw err; // unknown error, rethrow it
  }
}
```

The new class `PropertyRequiredError` is easy to use: we only need to pass the property name: `new PropertyRequiredError(property)`. The human-readable `message` is generated by the constructor.

Please note that `this.name` in `PropertyRequiredError` constructor is again assigned manually. That may become a bit tedious -- to assign `this.name = <class name>` in every custom error class. We can avoid it by making our own "basic error" class that assigns `this.name = this.constructor.name`. And then inherit all our custom errors from it.

Let's call it `MyError`.

Here's the code with `MyError` and other custom error classes, simplified:

```js run
class MyError extends Error {
  constructor(message) {
    super(message);
*!*
    this.name = this.constructor.name;
*/!*
  }
}

class ValidationError extends MyError { }

class PropertyRequiredError extends ValidationError {
  constructor(property) {
    super("No property: " + property);
    this.property = property;
  }
}

// name is correct
alert( new PropertyRequiredError("field").name ); // PropertyRequiredError
```

Now custom errors are much shorter, especially `ValidationError`, as we got rid of the `"this.name = ..."` line in the constructor.

## Wrapping exceptions

The purpose of the function `readUser` in the code above is "to read the user data", right? There may occur different kinds of errors in the process. Right now we have `SyntaxError` and `ValidationError`, but in the future `readUser` function may grow: the new code will probably generate other kinds of errors.

The code which calls `readUser` should handle these errors. Right now it uses multiple `if`s in the `catch` block, that check the class and handle known errors and rethrow the unknown ones.

The scheme is like this:

```js
try {
  ...
  readUser()  // the potential error source
  ...
} catch (err) {
  if (err instanceof ValidationError) {
    // handle validation errors
  } else if (err instanceof SyntaxError) {
    // handle syntax errors
  } else {
    throw err; // unknown error, rethrow it
  }
}
```

In the code above we can see two types of errors, but there can be more.

If the `readUser` function generates several kinds of errors, then we should ask ourselves: do we really want to check for all error types one-by-one every time?

Often the answer is "No": we'd like to be "one level above all that". We just want to know if there was a "data reading error" -- why exactly it happened is often irrelevant (the error message describes it). Or, even better, we'd like to have a way to get the error details, but only if we need to.

The technique that we describe here is called "wrapping exceptions".

1. We'll make a new class `ReadError` to represent a generic "data reading" error.
2. The function `readUser` will catch data reading errors that occur inside it, such as `ValidationError` and `SyntaxError`, and generate a `ReadError` instead.
3. The `ReadError` object will keep the reference to the original error in its `cause` property.

Then the code that calls `readUser` will only have to check for `ReadError`, not for every kind of data reading errors. And if it needs more details of an error, it can check its `cause` property.

Here's the code that defines `ReadError` and demonstrates its use in `readUser` and `try..catch`:

```js run
class ReadError extends Error {
  constructor(message, cause) {
    super(message);
    this.cause = cause;
    this.name = 'ReadError';
  }
}

class ValidationError extends Error { /*...*/ }
class PropertyRequiredError extends ValidationError { /* ... */ }

function validateUser(user) {
  if (!user.age) {
    throw new PropertyRequiredError("age");
  }

  if (!user.name) {
    throw new PropertyRequiredError("name");
  }
}

function readUser(json) {
  let user;

  try {
    user = JSON.parse(json);
  } catch (err) {
*!*
    if (err instanceof SyntaxError) {
      throw new ReadError("Syntax Error", err);
    } else {
      throw err;
    }
*/!*
  }

  try {
    validateUser(user);
  } catch (err) {
*!*
    if (err instanceof ValidationError) {
      throw new ReadError("Validation Error", err);
    } else {
      throw err;
    }
*/!*
  }

}

try {
  readUser('{bad json}');
} catch (e) {
  if (e instanceof ReadError) {
*!*
    alert(e);
    // Original error: SyntaxError: Unexpected token b in JSON at position 1
    alert("Original error: " + e.cause);
*/!*
  } else {
    throw e;
  }
}
```

In the code above, `readUser` works exactly as described -- catches syntax and validation errors and throws `ReadError` errors instead (unknown errors are rethrown as usual).

So the outer code checks `instanceof ReadError` and that's it. No need to list all possible error types.

The approach is called "wrapping exceptions", because we take "low level" exceptions and "wrap" them into `ReadError` that is more abstract. It is widely used in object-oriented programming.

## Summary

- We can inherit from `Error` and other built-in error classes normally. We just need to take care of the `name` property and don't forget to call `super`.
- We can use `instanceof` to check for particular errors. It also works with inheritance. But sometimes we have an error object coming from a 3rd-party library and there's no easy way to get its class. Then `name` property can be used for such checks.
- Wrapping exceptions is a widespread technique: a function handles low-level exceptions and creates higher-level errors instead of various low-level ones. Low-level exceptions sometimes become properties of that object like `err.cause` in the examples above, but that's not strictly required.
