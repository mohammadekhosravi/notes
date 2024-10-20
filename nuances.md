## Nuances That I Found While Study Typescript

1. object as a function paramerter:

```typescript
type Coords = { x: number; y: number };
function printCoords(coord: Coords) {}

// this gives error
printCoords({ x: 1, y: 1, z: 2 });

// but if you define a variable with excessive key value pair typescript doesn't care
const myCoords = { x: 1, y: 1, z: 1 };
printCoords(myCoords);
```

2. calling array functions on tuple type

```typescript
type HTTPResponse = [number, string];

// this throw error
const ok: HTTPResponse = [200, "OK", 2];

// this is allowed
const notFound: HTTPResponse = [400, "NotFound"];
notFound.push("This is allowed");
notFound.pop();
notFound.pop();
notFound.pop();
```
