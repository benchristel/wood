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
function App() {
    let name = "world";
    return (self) => (
        <div>
            <input
                type="text"
                value={name}
                onInput={({currentTarget: {value}}) => name = value}
            />
            <GreetingStopwatch name={name} />
        </div>
    );
}

function GreetingStopwatch(self) {
    let elapsedSeconds = 0;
    useInterval(self, () => elapsedSeconds++, 1000);
    
    // If the condition passed to `$remountIf` returns true on any render,
    // the component is unmounted and a new instance is mounted.
    self.$remountIf((oldProps, newProps) => oldProps.name !== newProps.name);

    return (self) => (
        <div>
            <p>Hello, {self.name}!</p>
            <p>`name` has been "{self.name}" for {elapsedSeconds}s.</p>
        </div>
    );
}
```

## Refs

### JavaScript

```jsx
function MeasuredSpan(self) {
    let span = {current: null};
    let lastMeasuredWidth = null;
    self.$afterRender(() => {
        const currentWidth = span.current?.clientWidth;
        if (currentWidth !== lastMeasuredWidth) {
            lastMeasuredWidth = currentWidth;
            self.onWidthChanged(currentWidth);
        }
    });
    return (self) => (
        <span ref={span}>
            {self.children}
        </span>
    );
}

function App(self) {
    let spanWidth = null;
    return (self) => (
        <div>
            <MeasuredSpan onWidthChanged={(w) => spanWidth = w}>
                This is some text
            </MeasuredSpan>
            <span>
                - The preceding text has a width of {spanWidth}
            </span>
        </div>
    );
}
```

### TypeScript

```tsx
interface Props {
    onWidthChanged: (pixels: number | null) => unknown;
}

const MeasuredSpan: Component<Props> = (self) => {
    let span = {current: null as null | HTMLSpanElement};
    let lastMeasuredWidth = null as number | null;
    self.$afterRender(() => {
        const currentWidth = span.current?.clientWidth;
        if (currentWidth !== lastMeasuredWidth) {
            lastMeasuredWidth = currentWidth;
            self.onWidthChanged(currentWidth);
        }
    });
    return (self) => (
        <span ref={span}>
            {self.children}
        </span>
    );
}

const App: Component = (self) => {
    let spanWidth = null as number | null;
    return (self) => (
        <div>
            <MeasuredSpan onWidthChanged={(w) => spanWidth = w}>
                This is some text
            </MeasuredSpan>
            <span>
                - The preceding text has a width of {spanWidth}
            </span>
        </div>
    );
}
```

## What is this wizardry?

There are some key differences between `wood` and React that enable `wood` components to be simpler.

- You'll notice in the examples above that many of the components consist
  of an outer function, which contains state and effects, and an inner function, which renders JSX. For components that have this structure,
  _the outer function is only ever called once_, to initialize the component before it mounts. The inner function is called for every render.
- Components only re-render if `self.$markForRerender()` is called—though, as described below, `wood` will call `$markForRerender()` for you in certain predictable cases. The preference for explicit rerendering affords a level of control that isn't easy to achieve in React, and lets you write highly optimized components if you need to.
- `self` is mutable. It always has the latest version of the props. This means that effect callbacks that refer to `self` will always see the current props, without any need for React's mechanism of specifying a dependency array.
- When a component renders JSX, `wood` inspects the JSX elements for callbacks. If it finds a callback that is _not_ one of the component's props,
it wraps the callback in a function like this:
  ```js
  () => {
    self.$markForRerender()
    callback()
  }
  ```
  This is why components can simply use variables to store state, rather than something like `useState` or `setState`. Any callbacks that might affect the component's state automatically trigger a re-render.

## Limitations of `wood` versus React

The simplifications described above impose some restrictions on what components can do. I believe this is a good thing. Compared to React, `wood` encourages components to have a single, focused purpose, and not to mix state, effects, and views unnecessarily.

For example, in React, components can respond to props changes by
re-running effects, without losing the state of the component. In `wood`,
components have only one way to re-run effects: they can unmount and remount, which resets their state.

In React, avoiding imperative interactions between props and state is considered a best practice. In `wood`, it's a requirement.
