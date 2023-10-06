# wood

Components, JSX, and hook-like encapsulated effects, minus the boilerplate.

## "Hello, World" Example

### JavaScript

```jsx
import {render} from "wood";

function Greeting(self) {
    return <p>Hello, {self.name}!</p>;
}

render(<Greeting name="world" />, document.body);
```

### TypeScript

```tsx
import {render} from "wood";

interface Props {
    name: string;
}

const Greeting: Component<Props> = (self) => {
    return <p>Hello, {self.name}!</p>;
}

render(<Greeting name="world" />, document.body);
```

## Counter Example (State)

In this and subsequent examples, the call to `render` is omitted for brevity.

### JavaScript

```jsx
function Counter() {
    let count = 0;
    return (self) => (
        <button onClick={() => count++}>
            The count is {count}
        </button>
    );
}
```

### TypeScript

```tsx
const Counter: Component = () => {
    let count = 0;
    return (self) => (
        <button onClick={() => count++}>
            The count is {count}
        </button>
    );
}
```

## Clock Example (Effects)

### JavaScript

```jsx
function Clock(self) {
    useInterval(self, () => {}, 1000);
    return (self) => (
        <p>{Date()}</p>
    );
}

function useInterval(self, callback, intervalMillis) {
    const interval = setInterval(() => {
        self.$markForRerender();
        callback();
    }, intervalMillis);
    self.$cleanup(() => clearInterval(interval))
}
```

### TypeScript

```tsx
const Clock: Component = (self) => {
    useInterval(self, () => {}, 1000);
    return (self) => (
        <p>{Date()}</p>
    );
}

function useInterval(
    self: Self,
    callback: () => unknown,
    intervalMillis: number,
) {
    const interval = setInterval(() => {
        self.$markForRerender();
        callback();
    }, intervalMillis);
    self.$cleanup(() => clearInterval(interval))
}
```

## Fetch Example (State and Effects)

### JavaScript

```jsx
function ApiVersion(self) {
    let data, error
    useFetch(self, "/api/version")
        .then((_data) => data = _data)
        .catch((_error) => error = _error);
    
    return (self) => {
        if (data) {
            return <p>{data.version}</p>;
        }
        if (error) {
            return <p>{String(error)}</p>;
        }
        return <p>loading...</p>;
    }
}

function useFetch(self, uri) {
    return fetch(uri)
      .then(response => response.json())
      .finally(() => self.$markForRerender())
}
```

## Resetting State when Props Change

### JavaScript

```jsx
function GreetingStopwatch(self) {
    let elapsedSeconds = 0;
    useInterval(self, () => elapsedSeconds++, 1000);
    
    // If the condition passed to `$remountIf` returns true on any render,
    // the component is unmounted and a new instance is mounted.
    self.$remountIf((oldSelf, newSelf) => oldSelf.name !== newSelf.name);

    return (self) => (
        <div>
            <p>Hello, {self.name}!</p>
            <p>`name` has been "{self.name}" for {elapsedSeconds}s.</p>
        </div>
    );
}
```