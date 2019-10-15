# Practical Advanced Typescript

## 1. Improve Readability with Typescript Numeric Seperators

When working with large number, such as the population of Africa, 1287269147, it would be really hard to tell at a glance how big is the number actually is. Outside the programming, we should use the seperator to make it easier for other humans to read, like 1,287,269,147. Now we can instantly tell that is approximately 1.2 billion.

Since Typescript 2.7, we can use underscore as numeric seperator in our code.

```javascript
const africaPop = 1_287_269_147;
```

It makes the number more readable but doesn't affect the output at all.

## 2. Make Typescript Class Usage Safer with Strict Property Initialization

Here I have a class Library with a single property titles, that's of type array of string. Then below, I create a new instance of that class. It is quite obvious that the title array has not been initialized and undefined.
However, somewhere else in my codebase, I try to use library instance to get titles from it and then filter through it.

```typescript
class Library {
  titles: string[];

  constructor() {}
}

const library = new Library();

// somtime later & elsewhere in our codebase

const shortTitles = library.titles.filter(
  title => title.length < 5
);
```

Now if I open ternimal to run the generated Javascript, I get the type error says

```bash
TypeError: Cannot read property 'filter' of undefined
```

In TypeScript 2.7, there's new tsconfig flag called `strictPropertyInitialization` that will warn us about these type of problem at compile time.

For it to work, we will alse need to enable the `strictNullCheck` flag.

```json
{
  "compilerOptions": {
    "strictPropertyInitialization": true,
    "strictNullCheck": true
  }
}
```

Now we have to either initialize the property directly or initialize them in constructor.

```typescript
class Library {
  titles: string[] = ['Harry Potter', 'The Lord of the Rings'];

  constructor() {}
}
```

```typescript
class Library {
  titles: string[];

  constructor() {
    this.titles = ['Harry Potter', 'The Lord of the Rings']
  }
}
```

Finally, we might be using a dependency injection library that we know will initialize all these classes' property at runtime. In that case, we can just start an exclaimation mark after the properties' name. This is called definite assignment operator. It a way of telling TypeScript that this property will definitely be initialized.

```typescript
class Library {
  titles!: string[];

  constructor() {}
}
```

Keep in mind, this operator doesn't help TypeScript think carefully about the potential unsafe usage of properties they add.

## 3. Use the JavaScript "in" operator for automatic type inference in TypeScript

I have two interfaces, one for admin and the other for normal user. And I have a function called redirect accept an argument either an admin or a user. If the user is admin, I want to redirect to a specific route and pass in the admin-only property. Otherwise, I want to use the normal route and pass in just email which is the user-only property.

```typescript
interface Admin {
  id: string;
  role: string;
}
interface User {
  email: string;
}

function redirect(usr: Admin | User) {
  if (/* user is admin */) {
    routeToAdminPage(usr.role);
  } else {
    routeToHomePage(usr.email);
  }
}
```

I can create custom type guard below, that will fix our problem.

```typescript
function redirect(usr: Admin | User) {
  if (isAdmin(usr)) {
    routeToAdminPage(usr.role);
  } else {
    routeToHomePage(usr.email);
  }
}

function isAdmin(usr: Admin | User): usr is Admin {
  return (<Admin>usr).role !== undefined;
}
```

But that might be to much when we create this function everytime we need TypeScript to infer types based on simple properies.

There is another option. The JavaScript `in` operator is useful for checking if certain properties exist on objects. Since TypeScript 2.7, the TypeScript start to infer the type of an object in a block that wrap the condition containing the `in` operator.

```typescript
function redirect(usr: Admin | User) {
  if ('role' in usr) {
    routeToAdminPage(usr.role);
  } else {
    routeToHomePage(usr.email);
  }
}
```

### 4. Automatically infer TypeScript type in switch statement.

Here I have a very simple Redux setup for a Todo application. There is a reducer function which accept an action and also the previous state. This state is designed to have a todos property that's an array of string.

```typescript
interface ITodoState {
  todos: string[];
}

function todoReducer(
  action: Action;
  state: ITodoState = { todos: [] }
): ITodoState {
  switch (action.type) {
    case "Add": {
      return {
        todos: [ ...state.todos, action.payload]
      }
    }
    case "Remove All": {
      return {
        todos: []
      }
    }
  }
}
```

To have a look of type `Action`, it's a very simple interface with a single property `type`. In redux, all of the actions must have at least the type. We also have a bunch of concrete action defined here that implement that `Action`. The `Add` action not only has `type` property, it also has a payload which represents the text of todo when want to add. The `RemoveAll` action intend to remove all the todos and it doesn't have payload.

```typescript
// todo.actions.ts
export interface Action {
  type: string;
}

export class Add implements Action {
  readonly type: string = "Add";
  constructor(public payload: string) {}
}

export class RemoveAll implements Action {
  readonly type: string = "Remove All";
}
```

Now we know that each action type has a unique string as its `type` property. So if we have switch statement, for TypeScript to be able to infer this class automatically for each string, two things need to be happen.

The first one is that the type property of these actions can't be a generic string anymore. If they're all string, TypeScript won't be able to tell them apart. There is a string literal type that allows us to set the type of a property to a very sepecfic string. To do it, we can just remove the string type from the `type` property.

```typescript
// todo.actions.ts
export interface Action {
  type: string;
}

export class Add implements Action {
  readonly type = "Add"; // Now it is sepecfic "Add" string
  constructor(public payload: string) {}
}

export class RemoveAll implements Action {
  readonly type = "Remove All";
}

export type TodoActions = Add | RemoveAll;
```

Very important to note here is that if I remove the `readonly` declaration from here, property `type` will revert back to being a generic string type. That because, by removing `readonly` flag, TypeScript can't make sure any more that the property won't be modified later.

```typescript
function todoReducer(
  action: TodoActions;
  state: ITodoState = { todos: [] }
): ITodoState {
  switch (action.type) {
    case "Add": {
      return {
        todos: [ ...state.todos, action.payload]
      }
    }
    case "Remove All": {
      return {
        todos: []
      }
    }
    default: {
      const x: never = action
    }
  }
}
```

If we don't wanna miss any case, we could just add the default case where simply assign the action to a variable of type never. Now if I comment the `Remove All` case, the variable `x` in default will gets an error, that's because the `never` sets that this value will nerver occur.

### 5. Create Explicit and Readable Type declarations with TypeScript mapped Type Modifiers

If I have an interface `IPet`, I could use mapped type modifiers to make all of properties readonly.

```typescript
interface IPet {
  name: string;
  age: number;
}

type ReadOnlyPet {
  readonly [K in keyof IPet]: IPet[K]
}
```

Type like this can be useful for assigning as a piece of state to my Redux app for example, because state need to be immutable.

What if the `IPet` interface add an optional property, and I want `ReadOnlyPet` to remove all of optionals. Since TypeScript 2.8, I can now add a minus sign before the signal I want to remove.
Meawhile, a plus sign is also added to be feature. Now we can be more explicit about what I'm adding and what I'm removing.

```typescript
interface IPet {
  name: string;
  age: number;
  favoritePark?: string;
}

type ReadOnlyPet {
  +readonly [K in keyof IPet]-?: IPet[K]
}
```

### 6. Use Types vs. Interfaces

Most of the times, types aliases in TypeScript are used to alias a more complex type, like union of other types. On the other hand, interfaces are used for more traditional object-oriented purpose where define the shape of an object and then use that as a contract for function parameters or classes to implement. This is probably the most obvious differences between types and interfaces.

```typescript
type Pet = IDog | ICat;

interface IAnimal {
  age: number;
  eat(): void;
  speak(): string;
}

function feedAnimal(animal: IAnimal) {
  animal.eat()
}

class Animal implements IAnimal {
  age = 0;

  eat() {
    console.log("nom..nom..");
  }

  speak() {
    return "nom..nom..";
  }
}
```

But type aliases and interfaces are also very similar in a variety of ways.

```typescript
interface IAnimal {
  age: number;
  eat(): void;
  speak(): string;
}

type AnimalTypeAlias = {
  age: number;
  eat(): void;
  speak(): string;
}

let animalInterface: IAnimal;
let animalTypeAlias: AnimalTypeAlias;

animalInterface = animalTypeAlias;
```

I can assign `animalTypeAlias` and `animalInterface` one to the other without TypeScript complaining. That's because TypeScript uses structural typing. As long as these two types have the same structure, it doesn't really matter that they are distinct types.

Type aliases, as per the name, can act as an alias for a more complex type, like a function or an array.

But I can also build the equivalent of type `Eat` and `AnimalList` using an Interface.

```typescript
type Eat = (food: string) => void;
type AnimalList = string[];

interface IEat {
  (food: string): void;
}

interface IAnimalList {
  [index: number]: string;
}
```

With type aliases, I can express a merge of different other types by means of intersection type. If I want to do with interface, although possible, I'd have to create a totally new interface to express that merge.

```typescript
type Cat = IPet & IFeline; // It's both Pet and Feline

interface ICat extends IPet, IFeline {

}

interface IPet {
  pose(): void;
}

interface IFeline {
  nightvision: boolean;
}
```

There is no difference between types aliases and interfaces when it comes to using them interchangeably. An interface can both extend interface and a type. A type can be a intersection of both an interface and another type. And even the class can implement both interface and type.

One of the biggest difference between them is while with a type, I can have a union of multiple other types. But if extends interface with a union type, the TypeScript should complains. That's because the interface is a specific contract. You can't have it be one thing or the another, same goes in Class.


```typescript
type PetType = IDog | ICat;

interface IPet extends PetType { // TypeScript complains

}

class Pet implements PetType { // TypeScript complains

}

interface IDog {}
interface ICat {}
```

Finally, another functional difference between interfaces and type aliases is if I have two interfaces with the same name, when used, they will be merged. This something that TypeScript do not support. If we have two same name type, the TypeScript should complains.

```typescript
interface Foo {
  a: string;
}

interface Foo {
  b: string;
}
```

Now consider an example, importing a jQuery to our code and I’m tring to extend it by adding a new function to it. You don't have to touch the library, you just need to create a interface locally with the same name of jQuery interface. Then they will be merged.

### Build self-referening type aliases in TypeScript

```typescript
interface TreeNode<T> {
  value: T;
  left: TreeNode<T>;
  right: TreeNode<T>;
}
```

### Use TypeScript "unknow" type to aviod runtime error


