# Strict Property Check

Here has an interface and a function

```typescript
interface Animal {
    speciesName: string
    legCount: number,
}

function serializeBasicAnimalData(a: Animal) {
    // something
}
```
If call 

```typescript
serializeBasicAnimalData({
    legCount: 65,
    speciesName: "weird 65-legged animal",
    specialPowers: "Devours plastic"
})
```

it will got the error.

But if call, it don't get the error:

```typescript
const weirdAnimal = {
    legCount: 65,
    speciesName: "weird 65-legged animal",
    specialPowers: "Devours plastic"
};
serializeBasicAnimalData(weirdAnimal);
```

The underlying cause here is Typescripts reliance on structural typing which is a lot better than the alternative which is Nominal typing but still has its problems. You can use a `StrictPropertyCheck` to force it comply the interface.

```typescript
type StrictPropertyCheck<T, TExpected, TError> = Exclude<keyof T, keyof TExpected> extends never ? {} : TError;

interface Animal {
    speciesName: string
    legCount: number,
}

function serializeBasicAnimalData<T extends Animal>(a: T & StrictPropertyCheck<T, Animal, "Only allowed properties of Animal">) {
    // something
}

var weirdAnimal = {
    legCount: 65,
    speciesName: "weird 65-legged animal",
    specialPowers: "Devours plastic"
};
serializeBasicAnimalData(weirdAnimal); // now correctly fails
```